# quadlet-radicale

Quadlet setup for the [Radicale](https://radicale.org) CalDAV/CardDAV server (`ghcr.io/kozea/radicale:stable`).

This project was created with the help of Claude Code and https://github.com/mkoester/quadlet-my-guidelines/blob/main/new_quadlet_with_ai_assistance.md.

## Files in this repo

| File | Description |
|---|---|
| `radicale.container` | Quadlet unit file |
| `radicale.env` | Default environment variables |
| `radicale.override.env.template` | Template for local overrides |
| `radicale.config.template` | Template for the Radicale application config |
| `radicale-backup.service` | Systemd service: copies CalDAV/CardDAV data and config via `rsync` |
| `radicale-backup.timer` | Systemd timer: triggers the backup daily |
| `migrate-to-radicale.md` | How to migrate from another DAV server |

## Setup

```sh
# 1. Create service user (regular user, home in /var/lib)
sudo useradd -m -d /var/lib/radicale -s /usr/sbin/nologin radicale

REPO_URL=https://github.com/mkoester/quadlet-radicale.git
REPO=~radicale/quadlet-radicale
```

```sh
# 2. Enable linger
sudo loginctl enable-linger radicale

# 3. Clone this repo into the service user's home
sudo -u radicale git clone $REPO_URL $REPO

# 4. Create quadlet, config, and data directories
sudo -u radicale mkdir -p ~radicale/.config/containers/systemd
sudo -u radicale mkdir -p ~radicale/config
sudo -u radicale mkdir -p ~radicale/data

# 5. Create Radicale application config from template
sudo -u radicale cp $REPO/radicale.config.template ~radicale/config/config
# Edit if needed (e.g. to adjust timezone or add logging):
sudo -u radicale nano ~radicale/config/config

# 6. Create .override.env from template and fill in required values
sudo -u radicale cp $REPO/radicale.override.env.template $REPO/radicale.override.env

# 7. Symlink all quadlet files from the repo
sudo -u radicale ln -s $REPO/radicale.container ~radicale/.config/containers/systemd/radicale.container
sudo -u radicale ln -s $REPO/radicale.env ~radicale/.config/containers/systemd/radicale.env
sudo -u radicale ln -s $REPO/radicale.override.env ~radicale/.config/containers/systemd/radicale.override.env

# 8. Reload and start
sudo -u radicale XDG_RUNTIME_DIR=/run/user/$(id -u radicale) systemctl --user daemon-reload
sudo -u radicale XDG_RUNTIME_DIR=/run/user/$(id -u radicale) systemctl --user start radicale

# 9. Verify
sudo -u radicale XDG_RUNTIME_DIR=/run/user/$(id -u radicale) systemctl --user status radicale
```

## Configuration

### Environment variables

`radicale.env` contains the defaults:

| Variable | Default | Description |
|---|---|---|
| `TZ` | `Europe/Berlin` | Container timezone |

To override any value locally, edit `radicale.override.env` in the repo clone:

```sh
sudo -u radicale nano ~radicale/quadlet-radicale/radicale.override.env
sudo -u radicale XDG_RUNTIME_DIR=/run/user/$(id -u radicale) systemctl --user restart radicale
```

### Radicale application config

The Radicale config lives at `~radicale/config/config` on the host and is mounted read-only into the container at `/etc/radicale/config`.

The template sets:

| Option | Value | Description |
|---|---|---|
| `server.hosts` | `0.0.0.0:5232` | Listen on all interfaces inside the container |
| `auth.type` | `http_x_remote_user` | Auth delegated to Caddy (basicauth) |
| `storage.filesystem_folder` | `/var/lib/radicale/collections` | CalDAV/CardDAV data location |

Caddy handles password verification and forwards the authenticated username in the `X-Remote-User` header. Radicale reads that header (`auth.type = http_x_remote_user`) to identify the user and serve their personal collections.

## Reverse proxy (Caddy)

Add a site block to your Caddyfile. Replace `radicale.example.com` and the credentials as needed:

```
radicale.example.com {
    basic_auth {
        import /etc/caddy/radicale-credentials
    }
    reverse_proxy localhost:5232 {
        # Forward the authenticated username so Radicale can identify the user
        header_up X-Remote-User {http.auth.user.id}
    }
}
```

Create the credentials file. Each line has the format `username <bcrypt-hash>`:

```sh
# Generate a bcrypt hash for each user
caddy hash-password

# Create the credentials file
sudo tee /etc/caddy/radicale-credentials << 'EOF'
username <bcrypt-hash>
EOF

# Restrict access: Caddy (root) reads it, radicale user can read it for backup
sudo chown root:radicale /etc/caddy/radicale-credentials
sudo chmod 640 /etc/caddy/radicale-credentials
```

After editing the Caddyfile, reload Caddy:

```sh
sudo systemctl reload caddy
```

### Backing up Caddy credentials

`/etc/caddy/radicale-credentials` is picked up by the radicale backup service (see [Backup](#backup) below).

## Backup

Radicale stores CalDAV/CardDAV data as plain files on disk, so the backup uses `rsync`. See the [general backup setup](https://github.com/mkoester/quadlet-my-guidelines#backup) in the guidelines for the one-time server-wide setup (group, backup user, SSH key).

```sh
# 1. Create backup staging directories (owned by radicale, readable by backup-readers group)
sudo mkdir -p /var/backups/radicale/data /var/backups/radicale/config /var/backups/radicale/caddy
sudo chown -R radicale:backup-readers /var/backups/radicale
sudo chmod -R 750 /var/backups/radicale

# 2. Symlink the backup service and timer from the repo
sudo -u radicale mkdir -p ~radicale/.config/systemd/user
sudo -u radicale ln -s $REPO/radicale-backup.service ~radicale/.config/systemd/user/radicale-backup.service
sudo -u radicale ln -s $REPO/radicale-backup.timer ~radicale/.config/systemd/user/radicale-backup.timer

# 3. Enable and start the timer
sudo -u radicale XDG_RUNTIME_DIR=/run/user/$(id -u radicale) systemctl --user daemon-reload
sudo -u radicale XDG_RUNTIME_DIR=/run/user/$(id -u radicale) systemctl --user enable --now radicale-backup.timer
```

### On the remote (backup) machine

```sh
rsync -az backupuser@radicale-host:/var/backups/radicale/ /path/to/local/backup/radicale/
```

This pulls `data/` (CalDAV/CardDAV collections), `config/` (Radicale application config), and `caddy/` (Caddy credentials) into the local backup directory.

## Notes

- Port `5232` is bound to `127.0.0.1` only — Caddy handles TLS termination and basicauth.
- CalDAV/CardDAV data is stored at `~radicale/data/` on the host.
- `AutoUpdate=registry` is enabled; activate the timer once to get automatic image updates:
  ```sh
  sudo -u radicale XDG_RUNTIME_DIR=/run/user/$(id -u radicale) systemctl --user enable --now podman-auto-update.timer
  ```
- To prune old images automatically, enable the system-wide prune timer (see [image pruning setup](https://github.com/mkoester/quadlet-my-guidelines#image-pruning) for the one-time system setup). Replace `30` with the desired retention period in days:
  ```sh
  sudo -u radicale XDG_RUNTIME_DIR=/run/user/$(id -u radicale) systemctl --user enable --now podman-image-prune@30.timer
  ```
