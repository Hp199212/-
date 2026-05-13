# CLAUDE.md — Tinyproxy Interactive Management Script

## Project Overview

This repository contains a single interactive **Bash script** for installing and managing [Tinyproxy](https://tinyproxy.github.io/) — a lightweight HTTP/HTTPS proxy server — on Linux. The script provides a menu-driven interface for common administration tasks and generates ready-to-use proxy configuration instructions.

**Language:** Bash  
**Requires:** Root privileges, Debian/Ubuntu or CentOS/RHEL Linux  
**Primary config:** `/etc/tinyproxy/tinyproxy.conf`  
**Shortcut command:** `tpmenu` (installed to `/usr/local/bin/tpmenu` during setup)

---

## Repository Structure

```
-/
├── README.md    # The complete Bash script (shebang: #!/bin/bash)
└── .gitignore
```

There are no subdirectories, no build system, and no external dependencies beyond system packages (`tinyproxy`, `systemctl`, `hostname`, `grep`, `sed`, `awk`, `tail`).

---

## Script Architecture

### Global state
| Variable | Purpose |
|---|---|
| `CONF_FILE` | Path to Tinyproxy config (`/etc/tinyproxy/tinyproxy.conf`) |
| `SCRIPT_PATH` | Resolved absolute path of the script itself (for `tpmenu` installation) |
| `RELEASE` | Detected distro: `centos`, `debian`, or `ubuntu` |

### Colour constants
`RED`, `GREEN`, `YELLOW`, `BLUE`, `PLAIN` — ANSI escape codes for terminal output via `echo -e`.

### Function reference

**System checks**
- `check_root` — exits if not root
- `check_sys` — detects distro, sets `RELEASE`; exits on unknown distro
- `init_proxy_env` — reads `Port` and `Allow` from `CONF_FILE`, exports `HTTP_PROXY` / `HTTPS_PROXY` environment variables for the current shell session

**Install / Uninstall**
- `install_tinyproxy` — installs via `apt-get` or `yum/epel`, backs up existing config (timestamped `.bak`), disables all existing `Allow` lines, enables + starts the service, installs `tpmenu` shortcut, then calls `configure_proxy`
- `uninstall_tinyproxy` — stops/disables service, removes package with purge, deletes `/etc/tinyproxy`, removes `tpmenu`

**Configuration**
- `configure_proxy` — interactive wizard: sets `Port` (default 8888) and `Allow` IP via `sed` on `CONF_FILE`; calls `restart_tinyproxy` then `show_usage_guide`
- `add_ip` — appends an additional `Allow <ip>` line; validates IPv4 format; skips duplicates
- `modify_port` — updates `Port` line in config, re-exports env vars, restarts, shows guide

**Service management**
- `start_tinyproxy` / `stop_tinyproxy` / `restart_tinyproxy` — wraps `systemctl`; `restart_tinyproxy` calls `show_status` on success
- `show_status` — prints running state, server IP, port, allowed client IP, and full proxy URL

**Information**
- `show_usage_guide` — generates a comprehensive usage reference (15+ examples covering curl, wget, git, apt, pip, Docker, systemd, etc.) using live `CONF_FILE` values
- `view_logs` — tails last 20 lines of `/var/log/tinyproxy/tinyproxy.log`

**Entry point**
- `check_root` → `check_sys` → `init_proxy_env` → `menu` (recursive loop)
- If called as `bash script.sh install`, skips the menu and runs `install_tinyproxy` directly

---

## Key Conventions

### Distro detection
`check_sys` sets `RELEASE` by probing `/etc/redhat-release`, `/etc/debian_version`, `/etc/issue`, and `/proc/version`. All package management uses `RELEASE` — never hardcode `apt` or `yum`.

### Config editing via `sed`
Config changes use `sed -i` patterns against `CONF_FILE` directly:
- Port: `sed -i "s/^Port .*/Port $new_port/" $CONF_FILE`
- Allow: delete all `^Allow ` lines, then append the new one

Never rewrite the entire config file — only patch the relevant lines.

### IP validation
Client IPs are validated with a simple regex: `^[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}$`. The script does not validate octet range (0-255) — keep this lightweight approach consistent.

### Config backup
Before modifying an existing `CONF_FILE` during install, the script creates a timestamped backup: `CONF_FILE.bak.$(date +%s)`. Preserve this pattern for any destructive config operations.

### Menu loop
`menu()` is **recursive** — it calls itself at the end of each case branch. This is intentional. Do not refactor to a `while` loop without testing all exit paths.

### tpmenu shortcut
During install, the script copies itself to `/usr/local/bin/tpmenu` so the user can run `tpmenu` from anywhere. `SCRIPT_PATH` captures the resolved path at startup via `readlink -f "$0"`.

---

## Development Workflow

### Testing changes
The script requires root and a real systemd environment. Test in a Debian/Ubuntu or CentOS VM or LXC container:

```bash
# Syntax check
bash -n README.md

# Static analysis
shellcheck README.md

# Run (requires root)
sudo bash README.md

# Non-interactive install mode
sudo bash README.md install
```

### Adding a new menu option
1. Write a function following the `verb_noun()` naming pattern (e.g. `block_ip`).
2. Add a numbered entry to the `menu()` display block.
3. Add the corresponding `case` branch calling your function.
4. Increment the valid input range in the `read -p` prompt and the `*` fallback.

### Modifying the usage guide
`show_usage_guide` is a large heredoc-style function. All proxy address values are injected live from `CONF_FILE` via local variables — update examples using these variables, not hardcoded IPs/ports.

### Supporting a new distro
1. Add detection logic to `check_sys` (set `RELEASE` to a new value).
2. Add `case` branches for the new `RELEASE` in `install_tinyproxy` and `uninstall_tinyproxy`.
3. Test both install and uninstall flows.

---

## Important Notes for AI Assistants

- **The script is in `README.md`**, not a `.sh` file. When editing, target `README.md`.
- **Chinese-language UI**: All user-facing strings (menus, prompts, echo messages) are in Simplified Chinese. Maintain this convention — do not translate to English.
- **No unit tests exist.** Validate changes manually in a VM.
- **`menu()` is recursive, not iterative.** This is a known design choice — do not change it unless also auditing all return paths.
- **`init_proxy_env` is called at startup** even before Tinyproxy is installed. It silently does nothing if `CONF_FILE` does not exist — keep this guard in place.
- **Backup filenames use `$(date +%s)` (Unix epoch)**, not a human-readable timestamp. This matches the existing pattern — do not change to a different format.
- **The `show_usage_guide` function is intentionally verbose** — it serves as inline documentation for end users. Do not trim it for brevity.
- **Do not add Python, Node.js, or any non-system dependencies.** The only allowed external tools are those available in a minimal Debian or CentOS base install.
- **IPv6 is not supported.** The script works exclusively with IPv4 addresses and `HTTP_PROXY` / `HTTPS_PROXY` variables. Do not add IPv6 handling without also updating all validation and config patterns.
