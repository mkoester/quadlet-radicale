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
| `radicale-backup.service` | Systemd service: copies CalDAV/CardDAV data via `rsync` |
| `radicale-backup.timer` | Systemd timer: triggers the backup daily |

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
| `auth.type` | `none` | Auth delegated to Caddy (basicauth) |
| `storage.filesystem_folder` | `/var/lib/radicale/collections` | CalDAV/CardDAV data location |

Since Radicale is only reachable via `127.0.0.1:5232` (bound to localhost), and Caddy sits in front enforcing basicauth, setting `auth.type = none` is safe.

## Reverse proxy (Caddy)

Add a site block to your Caddyfile. Replace `radicale.example.com` and the credentials as needed:

```
radicale.example.com {
    basicauth {
        # Generate a bcrypt hash: caddy hash-password
        username <bcrypt-hash>
    }
    reverse_proxy localhost:5232
}
```

After editing the Caddyfile, reload Caddy:

```sh
sudo systemctl reload caddy
```

## Backup

Radicale stores CalDAV/CardDAV data as plain files on disk, so the backup uses `rsync`. See the [general backup setup](https://github.com/mkoester/quadlet-my-guidelines#backup) in the guidelines for the one-time server-wide setup (group, backup user, SSH key).

```sh
# 1. Create backup staging directory (owned by radicale, readable by backup-readers group)
sudo mkdir -p /var/backups/radicale
sudo chown radicale:backup-readers /var/backups/radicale
sudo chmod 750 /var/backups/radicale

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
