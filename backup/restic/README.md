# Restic Snapshot Backup Suite

Enterprise-grade, automated snapshot backup routines for personal workstations using [Restic](https://restic.net/).

---

## Suite Components

- `restic-local-home-backup`: Backs up `$HOME` to local storage (e.g. secondary internal drive or USB disk).
- `restic-remote-home-backup`: Backs up `$HOME` to remote repository over SFTP / REST / SSH.
- `restic-local-system-backup`: Full root filesystem (`/`) backup with package list exports.
- `restic-remote-system-backup`: Remote snapshot backup of the root filesystem (`/`).
- `rm-restic-logs`: Cleanup script to purge old backup stdout/stderr logs.
- `restic-home-exclude`: Standard exclude file for user home backups (caches, node_modules, temp files).
- `restic-system-exclude`: Standard exclude file for root system backups (virtual filesystems, transient state).
- `repo.env.example`: Template for environment variables and repository credentials.

---

## Installation & Deployment

### 1. User Home Backups (`restic-*-home-backup`)
Install scripts and exclusion lists into `~/.local/bin/`:
```bash
mkdir -p ~/.local/bin
cp restic-*-home-backup restic-home-exclude rm-restic-logs ~/.local/bin/
chmod +x ~/.local/bin/restic-*-home-backup ~/.local/bin/rm-restic-logs
```

### 2. Root System Backups (`restic-*-system-backup`)
Install scripts and exclusion lists into `/usr/local/bin/`:
```bash
sudo cp restic-*-system-backup restic-system-exclude /usr/local/bin/
sudo chmod +x /usr/local/bin/restic-*-system-backup
```

---

## Configuration (`/etc/restic/*.repo.env`)

Each backup script expects a corresponding configuration file in `/etc/restic/`:

| Script | Configuration File |
| :--- | :--- |
| `restic-local-home-backup` | `/etc/restic/local.home.repo.env` |
| `restic-remote-home-backup` | `/etc/restic/remote.home.repo.env` |
| `restic-local-system-backup` | `/etc/restic/local.system.repo.env` |
| `restic-remote-system-backup` | `/etc/restic/remote.system.repo.env` |

### Environment Variables Template
Create `/etc/restic/<name>.repo.env` using `repo.env.example` as a base:
```bash
# Repository target (local directory, sftp, s3, or b2)
RESTIC_REPOSITORY="/mnt/backups/local/home"

# Path to the file containing the repo password
RESTIC_PASSWORD_FILE="/etc/restic/home.repo.key"

# Compression level (off, auto, max)
RESTIC_COMPRESSION="auto"

# Optional hostname override
RESTIC_HOST="archlinux"
```

Set strict permissions:
```bash
sudo chmod 600 /etc/restic/*.repo.env
sudo chmod 600 /etc/restic/*.repo.key
```

### Initializing a Repository
Before running any backup for the first time, initialize the target:
```bash
source /etc/restic/local.home.repo.env
restic -r "$RESTIC_REPOSITORY" --password-file "$RESTIC_PASSWORD_FILE" init
```

---

## Retention Policy & Maintenance

Every backup script automatically manages snapshot lifecycle and verifies repository health:
1. **Snapshot Creation**: Backs up target path with exclusions and tags (`home` or `system`).
2. **Snapshot Retention**: Prunes old snapshots keeping:
   - `7` daily snapshots
   - `4` weekly snapshots
   - `12` monthly snapshots
3. **Integrity Validation**: Runs `restic check` to ensure index and pack integrity.
