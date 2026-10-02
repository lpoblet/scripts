# AI Agent Guidelines & Repository Architecture

This document is the **single source of truth** for all AI coding assistants (Claude, Gemini, Copilot, Junie, etc.) operating in this `dotfiles` repository.

---

## 1. Context & Repository Architecture

### Directory Structure & GNU Stow
The repository uses [GNU Stow](https://www.gnu.org/software/stow/) to manage symlinks from package directories directly into `$HOME`:
```text
dotfiles/
├── .aiignore                       # AI context exclusion rules
├── .gitignore                      # Git ignore rules
├── AGENTS.md                       # Canonical agent instructions
├── DEPENDENCIES.md                 # System package mapping across distros
├── install.sh                      # Idempotent stow installation script
├── alacritty/                      # Alacritty terminal configs (~/.config/alacritty/)
├── arch/                           # Arch-specific bash environment (~/.bashrc, etc.)
├── debian/                         # Debian-specific bash environment
├── fedora/                         # Fedora-specific bash environment
├── bashAliases/                    # Common shell aliases (~/.bash_aliases)
├── git/                            # Git configuration (~/.gitconfig, ~/.gitignore_global)
├── hypr/                           # Hyprland ecosystem (~/.config/hypr/)
├── sway/                           # Sway WM config (~/.config/sway/)
├── waybar/                         # Waybar status bar (~/.config/waybar/)
├── swaync/                         # SwayNC notification center (~/.config/swaync/)
├── nvim/                           # Neovim / LazyVim (~/.config/nvim/)
├── vim/                            # Classic Vim configuration (~/.vimrc)
├── tmux/                           # tmux multiplexer (~/.config/tmux/)
├── btop/                           # btop system monitor (~/.config/btop/)
├── fastfetch/                      # fastfetch info tool (~/.config/fastfetch/)
├── lazygit/                        # lazygit TUI config (~/.config/lazygit/)
├── starship/                       # Starship prompt (~/.config/starship.toml)
├── scripts/                        # User helper & backup scripts (~/.local/bin or ~/scripts)
└── systemd/                        # User systemd service & timer units
```

### GNU Stow Conventions
- Each top-level directory corresponds to a stow package.
- Inside each stow package, file paths mirror their target location relative to `$HOME`.
  - Example: `nvim/.config/nvim/init.lua` -> symlinked to `~/.config/nvim/init.lua`.
  - Example: `git/.gitconfig` -> symlinked to `~/.gitconfig`.
- Never place absolute symlinks or hardcoded user paths inside tracked files when relative or environment-based references (`$HOME`, `~`) can be used.

---

## 2. Risk Mitigation & Secret Isolation

### Strict Rules for Secrets
- **Zero Secrets in Git**: Never hardcode private keys, API tokens, passwords, or machine-specific tokens in tracked files.
- **Local Overrides Pattern**:
  - **Git**: Global `git/.gitconfig` includes `~/.gitconfig.local` for private/work credentials. Use `git/.gitconfig.local.example` as a template.
  - **Alacritty**: `alacritty.toml` imports `~/.config/alacritty/alacritty-local.toml` for machine-specific display/font overrides. Use `alacritty-local.toml.example` as a template.
  - **Backups / Restic**: Scripts load credentials from `/etc/restic/*.repo.env` or local `.env` files. Templates reside in `scripts/scripts/backup/restic/repo.env.example`.
- **AI Ignore Policy**: Ensure `.aiignore` ignores all `.env`, `.key`, `.pem`, `.secrets`, and local override files.

---

## 3. Idempotence, Modularity & Safety Constraints

### Scripting Standards
- **Idempotence**: All setup and automation scripts must be safe to run multiple times without duplicating lines, clobbering configs, or causing unexpected state changes.
- **Dry-run Support**: Scripts performing operations against `$HOME` or system paths should support a `--dry-run` (`-n`) flag where applicable.
- **Modularity**: When editing configs, modify only the relevant stow package. Avoid coupling independent components.
- **Defensive Shell Scripting**:
  - Always use `set -euo pipefail` in Bash scripts where appropriate.
  - Quote all variables (`"$VAR"`).
  - Use `command -v <tool>` to verify dependencies before execution.

---

## 4. Bash Scripting Guidelines for AI

When generating, modifying, or refactoring Bash scripts in this repository, all AI agents must adhere to the following standards:

### 1. Invocation & CLI UX Conventions
- **No-Argument Behavior**:
  - When a script requires mandatory arguments or subcommands to function, executing it **without parameters** must display usage/help information and exit with a non-zero status code (`exit 1` or `exit 2`).
  - Scripts that have safe, non-destructive default actions when run with zero parameters (e.g., displaying current status) are permitted, but any script expecting user input or arguments must fail gracefully to usage output.
- **Help Flags (`-h`, `--help`)**:
  - All scripts must support `-h` and `--help` flags.
  - Explicitly requesting help must print formatted usage information to `stdout` and exit with code `0`.
- **Usage Format**:
  - Help text must include:
    1. A concise single-line description.
    2. Usage syntax line (`Usage: <script_name> [OPTIONS] <REQUIRED_ARGS>`).
    3. Options / flags section with short/long forms, descriptions, and default values.
    4. Practical, realistic invocation examples.
- **Dry-Run Mode (`-n`, `--dry-run`)**:
  - Any script that creates, deletes, moves files, syncs directories, or modifies system state must provide a dry-run flag (`-n` / `--dry-run`) showing proposed actions without applying them.
- **Option Parsing**:
  - Parse CLI arguments robustly using a `while [[ $# -gt 0 ]]` loop with a `case` statement (or `getopts` for short-only options).
  - Handle unexpected flags by printing an error message, displaying usage, and exiting with status `2`.

### 2. Defensive Shell Scripting & Robustness
- **Shebang**: Use `#!/usr/bin/env bash` (or `#!/bin/bash`).
- **Strict Mode**: Declare `set -euo pipefail` at the very top of every script to fail immediately on errors, unset variables, and failed pipeline stages.
- **Variable Quoting**: Always quote variable expansions (`"$VAR"`, `"${ARRAY[@]}"`) and command substitutions (`"$(cmd)"`) to avoid word splitting and glob hazards.
- **Dependency Checking**: Guard external dependencies using `command -v <tool> >/dev/null 2>&1` before invoking them, providing clear installation or package hints when missing.
- **Path Portability**:
  - Never hardcode user home directory paths (e.g., `/home/leo`).
  - Always use `$HOME` or standard XDG base directory variables with sensible fallbacks:
    - `${XDG_CONFIG_HOME:-$HOME/.config}`
    - `${XDG_DATA_HOME:-$HOME/.local/share}`
    - `${XDG_STATE_HOME:-$HOME/.local/state}`
    - `${XDG_CACHE_HOME:-$HOME/.cache}`

### 3. Error Handling, Traps & Concurrency
- **Cleanup Traps**: Use `trap cleanup EXIT INT TERM` whenever allocating temporary files (`mktemp`), managing PID files, or creating lock resources.
- **Atomic Locking**: Prevent concurrent/overlapping runs for cron, backup, or background tasks using `flock` or lockfile descriptors.
- **Explicit Exit Codes**:
  - `0`: Successful execution (or explicit `--help` invocation).
  - `1`: General runtime error or precondition failure.
  - `2`: Invalid command-line syntax, unknown options, or missing required arguments.
  - `130`: Terminated by SIGINT (`Ctrl+C`).

### 4. Output, Logging & Desktop Notifications
- **Stream Separation**: Send regular program data/pipeline output to `stdout`; send diagnostic logs, status updates, warnings, and usage errors to `stderr` (`>&2`).
- **TTY Awareness**: Verify terminal attachment (`[ -t 1 ]`) before emitting ANSI color escape codes or interactive progress indicators.
- **Desktop Notifications**: When calling `notify-send`, ensure it is guarded (`command -v notify-send >/dev/null 2>&1`) and suppress errors in headless/SSH environments (`notify-send ... 2>/dev/null || true`).

### 5. Standard Script Boilerplate
All new Bash scripts should follow this general structure:
```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_NAME="$(basename "$0")"

usage() {
    cat <<EOF
Usage: ${SCRIPT_NAME} [OPTIONS] <TARGET>

A short description of what this script accomplishes.

Arguments:
  TARGET                  Target path or resource name (required)

Options:
  -n, --dry-run           Show actions without making changes
  -v, --verbose           Enable verbose output
  -h, --help              Show this help message and exit

Examples:
  ${SCRIPT_NAME} /path/to/dir
  ${SCRIPT_NAME} --dry-run /path/to/dir
EOF
}

# Require parameters: if no arguments supplied, show usage and exit with error
if [[ $# -eq 0 ]]; then
    usage >&2
    exit 2
fi

DRY_RUN=false
VERBOSE=false
TARGET=""

while [[ $# -gt 0 ]]; do
    case "$1" in
        -h|--help)
            usage
            exit 0
            ;;
        -n|--dry-run)
            DRY_RUN=true
            shift
            ;;
        -v|--verbose)
            VERBOSE=true
            shift
            ;;
        -*)
            echo "Error: Unknown option '$1'" >&2
            usage >&2
            exit 2
            ;;
        *)
            if [[ -z "$TARGET" ]]; then
                TARGET="$1"
            else
                echo "Error: Unexpected argument '$1'" >&2
                usage >&2
                exit 2
            fi
            shift
            ;;
    esac
done

if [[ -z "$TARGET" ]]; then
    echo "Error: Missing required argument <TARGET>" >&2
    usage >&2
    exit 2
fi

# Main logic here
if [[ "$DRY_RUN" == true ]]; then
    echo "[DRY-RUN] Processing: $TARGET"
else
    echo "Processing: $TARGET"
fi
```

---

## 5. Multi-Distribution Compatibility

When adding packages, systemd units, or shell helpers, reference `DEPENDENCIES.md` to ensure cross-distribution support:
- **Arch Linux**: `pacman -S <package>`
- **Debian / Ubuntu**: `apt-get install <package>`
- **Fedora**: `dnf install <package>`

Keep package naming differences documented in `DEPENDENCIES.md`.

---

## 6. Automated Verification & Linting

Before completing code or configuration changes, validate using automated checks:
- **Shell Scripts**: Validate with `bash -n` and `shellcheck` (run `scripts/validate.sh`).
- **Python Scripts**: Validate with `python3 -m py_compile` and `ruff`/`flake8` (run `scripts/validate.sh`).
- **Lua / Neovim**: Validate with `stylua` or Lua syntax linters.
- **JSON / TOML / YAML**: Verify syntax valid with standard parsers (`python3 -m json.tool`, `tomllib`).
- **Validation Script**: Execute `./scripts/validate.sh` to run the comprehensive test suite across all repository files.
