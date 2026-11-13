# System Scripts & Automation Suite

Modular system utilities, snapshot backup automation, and background synchronization routines for personal workstation management.

---

## Repository Architecture

```text
scripts/
├── junie-plan-sync                     # One-way SSH sync of the current folder's .junie/plans to a remote host (never deletes)
├── obsidian-uri                        # Converts second-brain note paths to Obsidian URIs and copies to clipboard
├── pomodoro                            # Robust CLI Pomodoro countdown timer with notifications
├── reloadwaybar                        # Restarts the Waybar status bar process
├── backup/
│   ├── restic/                         # Restic deduplicated snapshot backup suite
│   │   ├── README.md                   # Detailed setup and environment guide for Restic
│   │   ├── repo.env.example            # Environment template for Restic repository credentials
│   │   ├── restic-home-exclude         # Exclude rules for user home backups
│   │   ├── restic-system-exclude       # Exclude rules for root system backups
│   │   ├── restic-local-home-backup    # Local backup routine for $HOME
│   │   ├── restic-remote-home-backup   # Remote backup routine for $HOME
│   │   ├── restic-local-system-backup  # Local backup routine for root (/)
│   │   ├── restic-remote-system-backup # Remote backup routine for root (/)
│   │   ├── restic-local-home-backup.service
│   │   ├── restic-local-home-backup.timer
│   │   ├── restic-local-system-backup.service
│   │   ├── restic-local-system-backup.timer
│   │   ├── restic-remote-home-backup.service
│   │   ├── restic-remote-home-backup.timer
│   │   ├── restic-remote-system-backup.service
│   │   └── restic-remote-system-backup.timer
│   └── rsync/                          # Knowledge base differential synchronization
│       ├── README.md                   # Sync configuration and systemd notes
│       ├── sb-local-backup             # Local second-brain vault mirror using rsync and flock
│       ├── sb-remote-backup            # Remote SSH second-brain vault sync with reachability check
│       ├── zk-local-backup             # Local Zettelkasten vault mirror
│       ├── zk-remote-backup            # Remote SSH Zettelkasten vault sync
│       ├── sb-local-backup.service
│       ├── sb-local-backup.timer
│       ├── sb-remote-backup.service
│       ├── sb-remote-backup.timer
│       ├── zk-local-backup.service
│       └── zk-local-backup.timer
```

---

## Dependencies & Requirements

Install required tools based on your Linux distribution:

| Tool | Arch Linux | Debian / Ubuntu | Fedora | Purpose |
| :--- | :--- | :--- | :--- | :--- |
| **Bash** | `bash` | `bash` | `bash` | Core scripting runtime |
| **Restic** | `pacman -S restic` | `apt install restic` | `dnf install restic` | Encrypted snapshot backups |
| **Rsync** | `pacman -S rsync` | `apt install rsync` | `dnf install rsync` | Differential vault sync |
| **flock** | `pacman -S util-linux` | `apt install util-linux` | `dnf install util-linux` | Concurrency lock management |
| **wl-clipboard** / **xclip** | `pacman -S wl-clipboard xclip` | `apt install wl-clipboard xclip` | `dnf install wl-clipboard xclip` | Clipboard integration (`wl-copy`, `xclip`) |
| **libnotify** | `pacman -S libnotify` | `apt install libnotify-bin` | `dnf install libnotify` | Desktop notifications (`notify-send`) |
| **OpenSSH** | `pacman -S openssh` | `apt install openssh-client` | `dnf install openssh-clients` | Remote backup transport |

---

## General CLI Utilities

### `obsidian-uri`
Converts a note file path within an Obsidian vault (default: `$HOME/Documents/second-brain/`) into an RFC 3986 compliant Obsidian URI (`obsidian://open?vault=...&file=...`) and copies it to the system clipboard (`wl-copy`, `xclip`, `xsel`, `pbcopy`). By default, the `vault` parameter is set to the URL-encoded full directory path of the vault.

- **Usage**:
  ```bash
  obsidian-uri [OPTIONS] <FILE>
  ```
- **Arguments**:
  - `<FILE>`: File name or path relative/absolute to the vault (required).
- **Options**:
  - `-m, --markdown`: Copy as Markdown link format `[Note Title](obsidian://open?...)`.
  - `-p, --print`: Print generated URI to `stdout` in addition to copying to clipboard.
  - `-v, --vault NAME`: Override vault name (default: URL-encoded full path of vault directory).
  - `-d, --vault-dir DIR`: Override vault directory path (default: `$HOME/Documents/second-brain`).
  - `-n, --dry-run`: Preview generated URI without modifying clipboard.
  - `-h, --help`: Display usage and options.
- **Examples**:
  ```bash
  # Copy raw URI for a note
  obsidian-uri "Fleeting Notes/Meeting.md"

  # Copy Markdown formatted link
  obsidian-uri -m "Projects/Migration Plan.md"

  # Pass full absolute path or tilde path
  obsidian-uri ~/Documents/second-brain/Daily/2026-10-02.md

  # Preview URI without copying to clipboard
  obsidian-uri --dry-run -m "Research/Quantum Physics.md"
  ```

---

### `junie-plan-sync`
Copies the current directory's Junie plan files (`.junie/plans/`) to a receiving host over SSH. Git is not required. The remote folder is always the current directory's path relative to `$HOME` (for example `scripts`, or `source/lab/proj`), and the plans are copied to `<remote folder>/.junie/plans/`.

The transfer is one-way and never deletes anything on the receiving host: `rsync` runs without `--delete`, and `--update` skips files that are newer on the receiver. Before anything is written, the script checks SSH reachability (`BatchMode=yes`, `ConnectTimeout=3`), lists the plan files and the full target path, and asks for confirmation. Only `y` or `yes` proceeds.

- **Usage**:
  ```bash
  junie-plan-sync [OPTIONS] [HOST]
  ```
- **Arguments**:
  - `[HOST]`: Receiving hostname (optional; prompted for when omitted).
- **Options**:
  - `-n, --dry-run`: Show the confirmation screen and run `rsync --dry-run`; creates nothing on the remote host.
  - `-h, --help`: Display usage and options.
- **Requirements**: The script prompts on stdin, so omitting `[HOST]` requires a terminal. The current directory must live under `$HOME`, and `.junie/plans/` must contain at least one file. Requires `rsync` and `ssh`, plus passwordless SSH key authentication to the receiving host.
- **Examples**:
  ```bash
  # Prompt for the host, then confirm
  junie-plan-sync

  # Sync to a known host, preview only
  junie-plan-sync --dry-run x1c3
  ```

---

### `pomodoro`
A defensive, POSIX-compliant CLI Pomodoro timer with atomic lockfile handling, in-terminal countdown, desktop notifications, and terminal bell alerts.

- **Usage**:
  ```bash
  pomodoro [OPTIONS] <MINUTES>
  ```
- **Arguments**:
  - `<MINUTES>`: Duration in minutes (positive integer or float).
- **Options**:
  - `-m, --message TEXT`: Custom desktop notification message (default: `Pomodoro timer finished!`).
  - `-t, --title TEXT`: Custom notification title (default: `Pomodoro`).
  - `-n, --dry-run`: Preview timer settings and durations without running.
  - `-h, --help`: Display usage and options.
- **Examples**:
  ```bash
  # Standard 25-minute Pomodoro session
  pomodoro 25

  # 5-minute short break with custom message
  pomodoro 5 -m "Break finished, get back to work!" -t "Break Timer"

  # Preview dry-run
  pomodoro --dry-run 15
  ```

---

### `reloadwaybar`
Restarts the Waybar status bar process. Useful after configuration changes to apply them immediately without logging out.

- **Usage**:
  ```bash
  reloadwaybar
  ```

---

## Restic Snapshot Backup Suite (`backup/restic/`)

Automated, encrypted, and deduplicated snapshot backups supporting local drives and remote servers.

### Configuration Prerequisites
Restic scripts read repository settings and credentials from `/etc/restic/*.repo.env` (permissions recommended: `chmod 600`):

1. Copy the template from `backup/restic/repo.env.example`:
   ```bash
   sudo mkdir -p /etc/restic
   sudo cp backup/restic/repo.env.example /etc/restic/local.home.repo.env
   ```
2. Populate the required environment variables:
   ```bash
   RESTIC_REPOSITORY="/mnt/backups/local/home"
   RESTIC_PASSWORD_FILE="/etc/restic/home.repo.key"
   RESTIC_COMPRESSION="auto"
   ```
3. Initialize the Restic repository (one-time setup):
   ```bash
   source /etc/restic/local.home.repo.env
   restic -r "$RESTIC_REPOSITORY" --password-file "$RESTIC_PASSWORD_FILE" init
   ```

### Scripts & Retention Policies
- **`restic-local-home-backup`**: Backs up user `$HOME` to local storage with exclusion rules (`restic-home-exclude`).
- **`restic-remote-home-backup`**: Backs up user `$HOME` to a remote server over SFTP/REST with reachability checks.
- **`restic-local-system-backup`**: Backs up the root filesystem (`/`) locally and exports installed package lists (`/var/backups/package-lists`).
- **`restic-remote-system-backup`**: Backs up root filesystem (`/`) to remote target.

**Snapshot Retention Policy**:
All backup routines enforce:
- **7 daily** snapshots
- **4 weekly** snapshots
- **12 monthly** snapshots
- Automated repository integrity validation (`restic check`) following pruning.

---

## Knowledge Base Rsync Sync Suite (`backup/rsync/`)

Differential synchronization scripts tailored for Markdown/Obsidian knowledge bases, including both Second-Brain and Zettelkasten vaults.

### `sb-local-backup`
- Mirrors `$HOME/Documents/second-brain/` to `$HOME/backups/second-brain/` using `rsync -avz --delete`.
- Uses atomic locking via `${XDG_RUNTIME_DIR:-/tmp}/obsidian_backup.lock` to prevent overlapping runs.

### `sb-remote-backup`
- Syncs the local Second-Brain vault to a remote machine over SSH (`rsync -avz --delete -e ssh`).
- Performs a non-interactive, low-timeout SSH reachability probe (`BatchMode=yes`, `ConnectTimeout=3`) before starting sync.
- Requires SSH key-based authentication configured with the remote host.

### `zk-local-backup`
- Mirrors `$HOME/Documents/zettelkasten/` to `$HOME/backups/zettelkasten/` using `rsync -avz --delete`.
- Uses atomic locking to prevent overlapping runs.

### `zk-remote-backup`
- Syncs the local Zettelkasten vault to a remote machine over SSH.
- Performs SSH reachability checks before starting sync.
- Requires SSH key-based authentication configured with the remote host.

---

## Systemd Automation

Each backup script ships with co-located `.service` and `.timer` unit files for scheduled background execution.

### User Units (Home Backups & Vault Sync)
User timers execute in the context of the logged-in user without requiring root privileges.

1. Symlink or copy units to `~/.config/systemd/user/`:
   ```bash
   mkdir -p ~/.config/systemd/user

   # Restic home backup
   ln -sf "$(pwd)/backup/restic/restic-local-home-backup.service" ~/.config/systemd/user/
   ln -sf "$(pwd)/backup/restic/restic-local-home-backup.timer" ~/.config/systemd/user/

   # Rsync vault sync
   ln -sf "$(pwd)/backup/rsync/sb-local-backup.service" ~/.config/systemd/user/
   ln -sf "$(pwd)/backup/rsync/sb-local-backup.timer" ~/.config/systemd/user/
   ln -sf "$(pwd)/backup/rsync/sb-remote-backup.service" ~/.config/systemd/user/
   ln -sf "$(pwd)/backup/rsync/sb-remote-backup.timer" ~/.config/systemd/user/
   ```

2. Reload and enable user timers:
   ```bash
   systemctl --user daemon-reload
   systemctl --user enable --now restic-local-home-backup.timer
   systemctl --user enable --now sb-local-backup.timer
   systemctl --user enable --now sb-remote-backup.timer
   ```

### System Units (Root System Backups)
Root system backups require elevated permissions to read all system files.

1. Copy system units to `/etc/systemd/system/`:
   ```bash
   sudo cp backup/restic/restic-local-system-backup.service /etc/systemd/system/
   sudo cp backup/restic/restic-local-system-backup.timer /etc/systemd/system/
   ```

2. Reload and enable system timers:
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable --now restic-local-system-backup.timer
   ```

3. Verify active timer schedules:
   ```bash
   systemctl list-timers --all
   systemctl --user list-timers --all
   ```
