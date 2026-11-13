# Container Backup Setup Guide

This document describes how to set up local restic backups inside Docker containers (Arch Linux and Fedora) using the backup scripts from this repository.

## Overview

Both the Arch (`lpic1-arch`) and Fedora (`lpic1-fedora`) containers from `~/source/lab/docker/` can run local restic backups. The scripts repo is mounted read-only at `/home/student/scripts` inside each container. A shared restic repository stores both home and system backups, distinguished by `--tag`.

## Prerequisites

- Docker and Docker Compose installed on the host.
- The lab repo at `~/source/lab/` with the `docker/` directory.
- The scripts repo at `/home/leo/scripts/`.

## Setup Steps

### 1. Configure the Scripts Repo Mount

Create or update `~/source/lab/docker/.env`:

```
SCRIPTS_PATH=/home/leo/scripts
```

This ensures the scripts repo is mounted at `/home/student/scripts` inside each container.

### 2. Start the Containers

```bash
cd ~/source/lab/docker
docker compose up -d arch fedora
```

Verify they are running:

```bash
docker ps --filter "name=lpic1-arch" --filter "name=lpic1-fedora"
```

### 3. Fix DNS Resolution (if needed)

If containers cannot resolve hostnames (pacman/dnf fails), add a nameserver:

```bash
docker exec lpic1-arch sudo bash -c 'echo "nameserver 8.8.8.8" >> /etc/resolv.conf'
docker exec lpic1-fedora sudo bash -c 'echo "nameserver 8.8.8.8" >> /etc/resolv.conf'
```

### 4. Install Restic

Install restic inside each container via the package manager:

```bash
# Arch
docker exec lpic1-arch sudo pacman -S --noconfirm restic

# Fedora
docker exec lpic1-fedora sudo dnf install -y restic
```

Verify:

```bash
docker exec lpic1-arch restic version
docker exec lpic1-fedora restic version
```

### 5. Initialize Shared Restic Repository

Inside each container, create the repository and credentials:

```bash
docker exec <container> bash -c '
  mkdir -p /home/student/backups/restic /home/student/.local/share/restic
  head -c 32 /dev/urandom | base64 > /home/student/.local/share/restic/repo.key
  chmod 600 /home/student/.local/share/restic/repo.key
  export RESTIC_PASSWORD_FILE=/home/student/.local/share/restic/repo.key
  restic init --repo /home/student/backups/restic
'
```

### 6. Create Config Files

Create `/etc/restic/` env files pointing both home and system backups to the same repository:

```bash
docker exec <container> sudo bash -c '
  mkdir -p /etc/restic
  cat > /etc/restic/local.home.repo.env << EOF
export RESTIC_REPOSITORY="/home/student/backups/restic"
export RESTIC_PASSWORD_FILE="/home/student/.local/share/restic/repo.key"
EOF
  cat > /etc/restic/local.system.repo.env << EOF
export RESTIC_REPOSITORY="/home/student/backups/restic"
export RESTIC_PASSWORD_FILE="/home/student/.local/share/restic/repo.key"
EOF
  chmod 644 /etc/restic/local.home.repo.env /etc/restic/local.system.repo.env
'
```

### 7. Prepare Backup Scripts

Copy scripts and exclude files to a writable location, removing `--dry-run` and fixing hardcoded paths:

```bash
docker exec <container> bash -c '
  mkdir -p /home/student/.local/bin /home/student/backups/package-lists

  # Copy exclude files
  cp /home/student/scripts/backup/restic/restic-home-exclude /home/student/.local/bin/
  cp /home/student/scripts/backup/restic/restic-system-exclude /home/student/.local/bin/

  # Adapt home backup script: fix hardcoded path, remove --dry-run
  sed "s|/home/leo/backups/package-lists|\$HOME/backups/package-lists|; s|--dry-run --verbose|--verbose|" \
    /home/student/scripts/backup/restic/restic-local-home-backup \
    > /home/student/.local/bin/restic-local-home-backup

  # Adapt system backup script: remove --dry-run
  sed "s|--dry-run --verbose|--verbose|" \
    /home/student/scripts/backup/restic/restic-local-system-backup \
    > /home/student/.local/bin/restic-local-system-backup

  chmod +x /home/student/.local/bin/restic-local-home-backup /home/student/.local/bin/restic-local-system-backup
'
```

### 8. Run Backups

**Home backup** (runs as student user):

```bash
docker exec <container> bash -c '
  export HOME=/home/student
  export PATH="$HOME/.local/bin:$PATH"
  /home/student/.local/bin/restic-local-home-backup
'
```

**System backup** (requires `sudo` for /etc access):

```bash
docker exec <container> sudo bash -c '
  export HOME=/home/student
  export PATH="/home/student/.local/bin:$PATH"
  /home/student/.local/bin/restic-local-system-backup
'
```

After the system backup, fix repository ownership so the student user can read all snapshots:

```bash
docker exec <container> sudo chown -R student:student /home/student/backups/restic /home/student/.cache/restic
```

### 9. Verify Snapshots

```bash
docker exec <container> bash -c '
  export RESTIC_REPOSITORY="/home/student/backups/restic"
  export RESTIC_PASSWORD_FILE="/home/student/.local/share/restic/repo.key"
  restic snapshots
'
```

Expected output shows both `home` and `system` tags in the same repository.

## Shared Repository Design

Both home and system backups use the same restic repository at `/home/student/backups/restic`. They are distinguished by tags:

- **Home backup**: `--tag home` — backs up `$HOME` with exclusions from `restic-home-exclude`.
- **System backup**: `--tag system` — backs up `/etc` and `/boot`.

This keeps the setup simple — one repository, one password, two tags. Pruning and integrity checks are performed per-tag by the backup scripts.

## Container vs VM Considerations

| Aspect | Docker Containers | VMs / Bare Metal |
|---|---|---|
| **Systemd** | Not running. No `.service`/`.timer` units. | Systemd available for automation. |
| **Persistence** | Home directory is a Docker volume. Tied to Docker. | Fully persistent filesystem. |
| **Restic install** | Runtime via `pacman`/`dnf`. Lost on container recreation. | Pre-installed or persistent package install. |
| **`/etc/restic/` config** | Requires `sudo`. Lost if container is recreated. | Standard location, persists across reboots. |
| **Scripts repo** | Read-only mount at `/home/student/scripts`. | Full read-write access. |
| **System backup** | Needs `sudo` to read `/etc/shadow`, `/etc/sudoers`, etc. | Run as root or with sudo. |
| **Backup targets** | Minimal `$HOME`, container-specific `/etc` and `/boot`. | Real user data and system config. |

## Limitations

- **No systemd timers**: Backups are run manually inside containers.
- **Runtime-only restic**: Restic is installed at runtime, not baked into Dockerfiles. Re-creating the container requires reinstallation.
- **Read-only scripts mount**: The scripts repo is mounted read-only, so scripts must be copied to a writable location for modifications.
- **DNS resolution**: Containers may need manual DNS configuration (`nameserver 8.8.8.8`) if the host's Docker DNS resolver doesn't propagate correctly.
