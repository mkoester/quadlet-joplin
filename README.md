# quadlet-joplin

Quadlet setup for [Joplin Server](https://joplinapp.org/help/api/references/server/) — self-hosted sync server for Joplin notes (`docker.io/joplin/server:latest`).

This project was created with the help of Claude Code and https://github.com/mkoester/quadlet-my-guidelines/blob/main/new_quadlet_with_ai_assistance.md.

## Files in this repo

| File | Description |
|---|---|
| `joplin.network` | Quadlet network shared by both containers |
| `joplin-db.container` | PostgreSQL database container |
| `joplin.container` | Joplin Server container |
| `joplin.env` | Default environment variables |
| `joplin.override.env.template` | Template for local overrides (base URL, DB password) |
| `joplin-backup.service` | Systemd service: backs up PostgreSQL database via pg_dump |
| `joplin-backup.timer` | Systemd timer: triggers the backup daily |

## Setup

```sh
# 1. Create service user (regular user, home in /var/lib)
sudo useradd -m -d /var/lib/joplin -s /usr/sbin/nologin joplin

REPO_URL=https://github.com/mkoester/quadlet-joplin.git
REPO=~joplin/quadlet-joplin
```

```sh
# 2. Enable linger
sudo loginctl enable-linger joplin

# 3. Clone this repo into the service user's home
sudo -u joplin git clone $REPO_URL $REPO

# 4. Create quadlet and data directories
sudo -u joplin mkdir -p ~joplin/.config/containers/systemd
sudo -u joplin mkdir -p ~joplin/db

# 5. Create .override.env from template and fill in required values
sudo -u joplin cp $REPO/joplin.override.env.template $REPO/joplin.override.env
sudo -u joplin nano $REPO/joplin.override.env

# 6. Symlink all quadlet files from the repo
sudo -u joplin ln -s $REPO/joplin.network ~joplin/.config/containers/systemd/joplin.network
sudo -u joplin ln -s $REPO/joplin-db.container ~joplin/.config/containers/systemd/joplin-db.container
sudo -u joplin ln -s $REPO/joplin.container ~joplin/.config/containers/systemd/joplin.container
sudo -u joplin ln -s $REPO/joplin.env ~joplin/.config/containers/systemd/joplin.env
sudo -u joplin ln -s $REPO/joplin.override.env ~joplin/.config/containers/systemd/joplin.override.env

# 7. Reload and start
sudo -u joplin XDG_RUNTIME_DIR=/run/user/$(id -u joplin) systemctl --user daemon-reload
sudo -u joplin XDG_RUNTIME_DIR=/run/user/$(id -u joplin) systemctl --user start joplin-db joplin

# 8. Verify
sudo -u joplin XDG_RUNTIME_DIR=/run/user/$(id -u joplin) systemctl --user status joplin-db joplin
```

## Configuration

`joplin.env` contains non-sensitive defaults:

| Variable | Default | Description |
|---|---|---|
| `APP_PORT` | `22300` | Port the Joplin Server listens on |
| `DB_CLIENT` | `pg` | Database backend |
| `POSTGRES_HOST` | `systemd-joplin-db` | Hostname of the DB container (default quadlet name for `joplin-db.container`) |
| `POSTGRES_PORT` | `5432` | PostgreSQL port |
| `POSTGRES_DATABASE` | `joplin` | Database name (used by Joplin Server) |
| `POSTGRES_DB` | `joplin` | Database name (used by PostgreSQL to create the DB on first run) |
| `POSTGRES_USER` | `joplin` | Database user |

`joplin.override.env` (created from the template) holds instance-specific and sensitive values:

| Variable | Description |
|---|---|
| `APP_BASE_URL` | Full public URL of your Joplin Server, e.g. `https://joplin.example.com` |
| `POSTGRES_PASSWORD` | PostgreSQL password — set the same value in both containers |

## First login

Default admin credentials are `admin@localhost` / `admin`. Change the password immediately after first login. It is recommended to create a separate non-admin user for client synchronisation.

## Reverse proxy (Caddy)

Add the following snippet to your Caddyfile:

```
@joplin host joplin.my_domain.tld
handle @joplin {
    reverse_proxy localhost:22300
}
```

And add a DNS A/CNAME record for `joplin.my_domain.tld` pointing to your server.

## UID verification

The `joplin.container` assumes UID 1001 for the `joplin` user inside the image (the `node` base image occupies UID 1000). Verify before starting:

```sh
podman inspect docker.io/joplin/server:latest --format '{{.Config.User}}'
podman run --rm --entrypoint grep docker.io/joplin/server:latest joplin /etc/passwd
```

If the UID differs, update `UserNS`, `User`, and `Group` in `joplin.container` accordingly.

Similarly for `joplin-db.container` (postgres:16, expected UID 999):

```sh
podman run --rm --entrypoint grep docker.io/postgres:16 postgres /etc/passwd
```

If the UID differs, update `UserNS`, `User`, and `Group` in `joplin-db.container` and re-chown the data directory:

```sh
sudo -u joplin podman unshare chown -R 999:999 ~joplin/db
```

## Backup

The backup runs `pg_dump` inside the database container and writes a SQL dump to `/var/backups/joplin/`. A remote machine pulls via `rsync` over SSH using the shared `backupuser`. See the [general backup setup](https://github.com/mkoester/quadlet-my-guidelines#backup) for the one-time server-wide setup (group, backup user, SSH key).

```sh
# 1. Create backup staging directory (owned by joplin, readable by backup-readers group)
sudo mkdir -p /var/backups/joplin
sudo chown joplin:backup-readers /var/backups/joplin
sudo chmod 750 /var/backups/joplin

# 2. Symlink the backup service and timer from the repo
sudo -u joplin mkdir -p ~joplin/.config/systemd/user
sudo -u joplin ln -s $REPO/joplin-backup.service ~joplin/.config/systemd/user/joplin-backup.service
sudo -u joplin ln -s $REPO/joplin-backup.timer ~joplin/.config/systemd/user/joplin-backup.timer

# 3. Enable and start the timer
sudo -u joplin XDG_RUNTIME_DIR=/run/user/$(id -u joplin) systemctl --user daemon-reload
sudo -u joplin XDG_RUNTIME_DIR=/run/user/$(id -u joplin) systemctl --user enable --now joplin-backup.timer
```

### On the remote (backup) machine

```sh
rsync -az backupuser@joplin-host:/var/backups/joplin/ /path/to/local/backup/joplin/
```

## Notes

- Port `22300` is bound to `127.0.0.1` only — place a reverse proxy in front for external access.
- PostgreSQL data is stored at `~joplin/db/` on the host.
- Notes content is stored in the PostgreSQL database — no additional volume is needed for the Joplin Server container.
- `AutoUpdate=registry` is enabled; activate the timer once to get automatic image updates:
  ```sh
  sudo -u joplin XDG_RUNTIME_DIR=/run/user/$(id -u joplin) systemctl --user enable --now podman-auto-update.timer
  ```
- To prune old images automatically, enable the system-wide prune timer (see [image pruning setup](https://github.com/mkoester/quadlet-my-guidelines#image-pruning) for the one-time system setup). Replace `30` with the desired retention period in days:
  ```sh
  sudo -u joplin XDG_RUNTIME_DIR=/run/user/$(id -u joplin) systemctl --user enable --now podman-image-prune@30.timer
  ```
