# Second-Brain Rsync Vault Synchronization

Differential file synchronization scripts designed for Obsidian vaults and Markdown knowledge bases.

---

## Included Scripts

- `sb-local-backup`: Mirrors local Obsidian vault (`$HOME/Documents/second-brain/`) to local backup directory (`$HOME/backups/second-brain/`) using `rsync -avz --delete` and `flock`.
- `sb-remote-backup`: Syncs local vault to a remote device over SSH with pre-flight network reachability testing.
- `obsidian-uri`: (Located in root) Converts a note file path within the vault into an Obsidian URI or Markdown link and copies it to the clipboard for referencing notes across vaults or apps.

---

## Configuration & Prerequisites

### 1. Local Backup Setup
Edit `sb-local-backup` if your vault or backup paths differ from defaults:
- `SOURCE_DIR`: Default is `"$HOME/Documents/second-brain/"`
- `BACKUP_DIR`: Default is `"$HOME/backups/second-brain/"`

### 2. Remote Backup Setup
`sb-remote-backup` requires passwordless SSH key authentication to the target machine:
1. Generate an SSH key if needed:
   ```bash
   ssh-keygen -t ed25519 -C "obsidian-sync"
   ```
2. Copy the public key to the remote host:
   ```bash
   ssh-copy-id user@remote_host
   ```
3. Edit the target settings in `sb-remote-backup`:
   ```bash
   REMOTE_USER="leo"
   REMOTE_HOST="X1C3"
   REMOTE_DIR="/home/leo/Documents/second-brain/"
   ```

---

## Systemd Scheduling

Symlink the user service and timer definitions from `sysdunits/` into `~/.config/systemd/user/`:

```bash
mkdir -p ~/.config/systemd/user
ln -sf "$(pwd)/../../sysdunits/sb-local-backup.service" ~/.config/systemd/user/
ln -sf "$(pwd)/../../sysdunits/sb-local-backup.timer" ~/.config/systemd/user/
ln -sf "$(pwd)/../../sysdunits/sb-remote-backup.service" ~/.config/systemd/user/
ln -sf "$(pwd)/../../sysdunits/sb-remote-backup.timer" ~/.config/systemd/user/
```

Enable and start timers:
```bash
systemctl --user daemon-reload
systemctl --user enable --now sb-local-backup.timer
systemctl --user enable --now sb-remote-backup.timer
```

Check schedule status:
```bash
systemctl --user list-timers --all | grep sb-
```
