# Omnomarchy (CachyOS + Hyprland) — System Design

> Goal: Create a modular, secure, developer‑friendly Wayland desktop based on CachyOS with Hyprland, using Dotter for configuration management, systemd user units for process control, and Git + scripts for updates and backups.

---

## 1) Scope & Objectives

**In Scope**

* Layered, modular config architecture using Dotter for templating and linking.
* Opinionated Hyprland ecosystem (Hyprland, Waybar, Hyprlock, Hypridle, Hyprpaper).
* Secure defaults (ufw, SSH key‑based auth, restic backups).
* Shell‑independent startup using systemd --user units.
* Developer stack: VSCodium, Ghostty/Kitty, zsh + starship + zoxide + fzf + ripgrep + btop.
* User override system that allows safe customization without breaking updates.

**Out of Scope**

* Full system installer or OS provisioning (CachyOS base assumed installed).
* Centralized configuration management (no Nix/Ansible).

**Non‑goals**

* Hidden complexity or heavy abstractions. Simplicity and transparency are key.

---

## 2) Key Design Decisions

* **Base OS**: CachyOS (pacman, performance‑tuned kernel, Btrfs).
* **Compositor**: Hyprland + Hyprlock + Hypridle + Hyprpaper.
* **Display/Login**: greetd + tuigreet -> launches Hyprland directly (no shell rc).
* **Configuration Engine**: Dotter for layering (L0 core -> L1 profile -> L2 host -> L3 user).
* **Process supervision**: systemd user units.
* **Firewall**: ufw (nftables backend).
* **Portal**: xdg‑desktop‑portal‑hyprland; env via ~/.config/environment.d.
* **Editor**: VSCodium.
* **Autostart**: systemd user units (Waybar, hypridle, etc.).
* **Ownership**: Only manage files owned by Omnomarchy (never overwrite user files).
* **Updates**: Git pull + migration scripts + Dotter apply.
* **Backups**: restic with timers and encryption.

---

## 2.5) Filesystem Layout & Names

* Repo folder -> `~/.local/src/omnom/` (Git working copy)
* CLI -> `nom` (symlink to `~/.local/src/omnom/omctl` in `~/.local/bin/`)
* Runtime state -> `~/.config/omnom/state.json`
* Project config (owner‑shipped) -> inside the repo (`dotter/`, `pkgs/`, `scripts/`)
* User overrides -> `~/.config/dotter/local.toml` and user‑owned fragments (for example `~/.config/hypr/overrides.conf`, `~/.config/waybar/config.d/*.json`)
* Environment -> `~/.config/environment.d/*.conf`

Add to PATH via environment.d so shells are not a dependency:

```
# ~/.config/environment.d/50-omnom.conf
PATH=$HOME/.local/bin:$PATH
OMNOM_PATH=$HOME/.local/src/omnom
```

---

## 3) Architecture Overview

```
User CLI: nom (symlink) -> omctl (bash wrapper)
  |
  |-- plan/apply/update -> Dotter (link + template configs)
  |-- feature set -> switch Dotter tags (modules)
  |
  +-> post-apply hooks -> systemd --user restart (only changed units)

Layers:
  L0 core     (dotter/global.toml)
  L1 profile  (dotter/profiles/*.toml)
  L2 host     (dotter/hosts/*.toml)
  L3 user     (~/.config/dotter/local.toml)

Packages (modules): hyprland, waybar, ufw, vscodium, devshell, terminals, keys-agent,
                    backup-restic, snapper

Updates: git pull -> migrations -> dotter apply -> restart affected units
```

**Why this is resilient**

* Shell changes cannot break the environment.
* User overrides are isolated and always win.
* Dotter manages ownership of config files, preventing clobbering.

---

## 4) Repository Structure

```
~/.local/src/omnom/
|- omctl                        # CLI script (invoked via `nom` symlink)
|- dotter/
|  |- global.toml              # core config + default packages
|  |- profiles/                # presets (for example cachyos-laptop, gpu-amd)
|  |- hosts/                   # optional host configs
|  |- user.local.toml.example  # user overrides template
|- pkgs/                       # Dotter packages (modules)
|  |- hyprland/
|  |  |- hyprland.conf
|  |  |- conf.d/00-base.conf
|  |  |- hypridle.conf
|  |  |- targets.toml
|  |- waybar/
|  |  |- config.base.jsonc
|  |  |- waybar-launch
|  |  |- systemd/waybar.service
|  |  |- targets.toml
|  |- ufw/...
|  |- vscodium/...
|  |- devshell/...
|  |- terminals/...
|  |- keys-agent/...
|  |- backup-restic/...
|  |- snapper/...
|- scripts/
|  |- post-apply.sh             # restart changed units, enable ufw, etc.
|  |- migrate.d/                # migration scripts
|  |- dotter-diff.sh            # dry-run diff plan
|- docs/README.md               # user guide

~/.config/omnom/
|- state.json                   # applied migrations, generation tracking

~/.local/bin/
|- nom -> ~/.local/src/omnom/omctl  # convenience symlink
```

---

## 5) Layering and Override Model

| Layer  | Purpose         | Example                               |
| ------ | --------------- | ------------------------------------- |
| **L0** | Core defaults   | `dotter/global.toml`                  |
| **L1** | Profile presets | `dotter/profiles/cachyos-laptop.toml` |
| **L2** | Host-specific   | `dotter/hosts/my-laptop.toml`         |
| **L3** | User overrides  | `~/.config/dotter/local.toml`         |

* Each layer merges sequentially; later layers override previous ones.
* Dotter handles file ownership via symlinks.
* Users edit only their local.toml or user‑owned fragments (for example `~/.config/hypr/overrides.conf`).

---

## 6) Module Contract (Dotter Package)

Each module/package contains:

* `targets.toml` -> maps files to destinations.
* Files under `pkgs/<name>/` (configs, scripts, systemd units, env files).
* Optional post‑apply actions defined in `scripts/post-apply.sh`.

Example: pkgs/waybar/targets.toml

```toml
["config.base.jsonc"]
source = "config.base.jsonc"
target = "~/.local/share/omnomarchy/waybar/config.base.jsonc"
type   = "link"

["waybar-launch"]
source = "waybar-launch"
target = "~/.local/bin/waybar-launch"
type   = "link"
chmod  = 755

["systemd/waybar.service"]
source = "systemd/waybar.service"
target = "~/.config/systemd/user/waybar.service"
type   = "link"
```

---

## 7) User Override Patterns

* Hyprland -> user overrides in `~/.config/hypr/overrides.conf`.
* Waybar -> additional fragments in `~/.config/waybar/config.d/*.json`.
* Environment -> user‑only drop‑ins in `~/.config/environment.d/99-local.conf`.
* Dotter -> user settings and module toggles in `~/.config/dotter/local.toml`.

Omnomarchy never touches user‑owned files.

---

## 8) CLI Design (nom / omctl)

**Commands**

* `nom plan` -> run Dotter in dry‑run/diff mode.
* `nom apply` -> apply Dotter config; execute post hooks.
* `nom update` -> git pull + migrations + apply.
* `nom feature set <cap> <module>` -> toggle module in Dotter config.
* `nom rollback` -> revert to previous state via Btrfs snapshot.

(You can still call the underlying script directly as `omctl`.)

**Workflow**

1. User edits `local.toml` or runs `nom feature set`.
2. Run `nom plan` to preview changes.
3. Run `nom apply` to update config safely.
4. `post-apply.sh` restarts only affected units.

---

## 9) State and Migrations

* `~/.config/omnom/state.json` tracks applied migrations and last generation.
* Migration scripts live in `scripts/migrate.d/` and are idempotent.
* Migrations update config structures, rename units, or move files.

---

## 10) Security Model

* Firewall -> ufw default deny incoming; allow outgoing.
* SSH -> key‑only authentication; no root login.
* Keys‑Agent -> auto‑generate ed25519 + GPG keys; integrate with gpg‑agent.
* Backups -> restic + systemd timer + encryption.
* Least privilege -> omctl runs as user except pacman/ufw actions.

---

## 11) Update & Backup Flow

**Update (owner)**

1. `git pull` in `~/.local/src/omnom`
2. Run migrations
3. `nom apply` (Dotter apply + post hooks)
4. Post hooks restart changed units

**Backup (restic)**

* `restic-backup.timer` triggers `restic-backup.service` daily.
* Encrypted backups to user‑configured target.
* Include both `~/.local/src/omnom` and `~/.config/omnom` in policy.

**Rollback**

* Create a Snapper snapshot before `nom apply`.
* Restore with `nom rollback` or Snapper GUI.

---

## 12) Testing & QA

* Static lint -> shellcheck, shfmt.
* Integration -> fresh CachyOS VM, `nom apply`, Hyprland session boots.
* Regression -> verify systemd units, ufw, Waybar, restic.

---

## 13) Roadmap

**v0.1.0**
Core engine, Hyprland, Waybar, ufw, VSCodium, devshell, backups, snapper, CLI basics.

**v0.2.0**
Keys‑agent, terminal modules (Ghostty/Kitty), migrations.

**v0.3.0**
Alternative bar (HyprPanel), NVIDIA profile, AppArmor baseline.

**v1.0.0**
CI, testing, polished documentation, stable upgrade path.

---

## 14) Philosophy

Omnomarchy aims to be Nix‑like in modularity but Arch‑simple in spirit:

* Human‑readable TOML configs.
* No hidden state.
* Predictable layering.
* Respect for user space.
* Strong defaults; safe overrides.

It should always be possible to fix, override, or understand your setup with just:

```
nom plan
nom apply
```

Nothing more.
