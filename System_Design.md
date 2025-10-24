# Omarch-Alt (CachyOS + Hyprland) - System Design

> Goal: Deliver an opinionated, **Hyprland-first** desktop on **CachyOS** that is
> **secure by default**, **developer-ready**, and **user-adjustable** - while
> remaining **owner-updatable** without trampling user changes.

---

## 1) Scope & Objectives

**In scope**
- User-space configuration and lifecycle: Hyprland, Waybar, portals, user services.
- Security posture: ufw (deny-incoming), optional SSH key-only, AppArmor (later).
- Developer stack: VSCodium, terminals (Ghostty/Kitty), zsh + starship + zoxide + fzf, ripgrep, btop.
- Backup & safety: restic (encrypted, timer), Btrfs snapshots via snap-pac (+ docs).
- Update model: owner ships new defaults/modules; users keep overrides; **migrations** carry changes forward safely.
- Shell-agnostic: no reliance on `.bashrc`/`.zshrc`; everything starts via systemd user or login manager.

**Out of scope (initial)**
- Full OS installer; CachyOS base is assumed present.
- Heavy config management (Nix/Home-Manager/Ansible) - we use a small, auditable bash toolchain.

**Non-goals**
- Magic that hides state. We prefer explicit, documented steps with observable logs and diffs.

---

## 2) Key Decisions

- **Base**: CachyOS (repos + kernel) with pacman.
- **Compositor**: Hyprland; companion tools `hypridle`, `hyprlock`, `hyprpaper`.
- **Login**: **greetd + tuigreet** -> starts Hyprland directly (no shell RC required).
- **Bar**: Waybar (module can be swapped with HyprPanel later).
- **Firewall**: ufw (nftables backend).
- **Portal**: `xdg-desktop-portal-hyprland`; env set via `environment.d`.
- **Editor**: VSCodium (from CachyOS/Arch repo).
- **Autostart**: systemd **user** units; bind to `graphical-session.target`.
- **Layering**: L0 core -> L1 profile -> L2 host -> L3 user (last writer wins).
- **Ownership**: We only own files we create; if we don’t own it, we never clobber.

---

## 3) Architecture (at a glance)

```
                ---------------------------------------------------
                |                   User CLI                       |
                |                    omctl                         |
                |-------^-------------------^-----------------------
                        |                   |
                 plan/apply/update     feature set
                        |                   |
              ---------------------   -------------
              |   Renderer        |   | Module Res |
              | (merge + stage)   |   | (provides/ |
              |-------^------------   | conflicts) |
                      |               |-----^-------
         L0 core  L1 profile  L2 host  L3 user      modules/*/module.yaml
            |         |          |        |              (files/units/env)
            |----------------------------------->  render/$GEN/...  ->  symlinks -> live ~/.config, /etc
                                                          |
                                                  systemd --user restart (only if changed)
```

**Why this resists breakage**
- No lifecycle logic in user shells; changing shells does not impact startup or updates.
- User overrides reside in L3 and always win; owner updates land in L0/L1 via migrations.

---

## 4) Repository Layout

```
omarch-alt/
|- omctl                      # main CLI (bash, ~200-300 LOC)
|- config.yaml                # L0: defaults + feature selection
|- profiles/                  # L1: selectable presets (e.g., cachyos-laptop)
|  |- cachyos-laptop.yaml
|  |- gpu-amd.yaml
|- hosts/                     # L2: per-machine pins (optional)
|  |- <hostname>.yaml
|- user/                      # L3: user overrides (never overwritten)
|  |- overrides.yaml
|- modules/                   # swappable parts (by capability)
|  |- hyprland/
|  |  |- module.yaml
|  |  |- files/...              # hyprland.conf, conf.d/*, hypridle.conf, etc.
|  |- waybar/
|  |  |- module.yaml
|  |  |- files/...              # config.base.json, style.css, systemd unit
|  |- ufw/...
|  |- vscodium/...
|  |- devshell/...              # zsh, starship, zoxide, fzf, ripgrep, btop
|  |- terminals/...             # ghostty/kitty
|  |- keys-agent/...            # SSH ed25519 + GPG + gpg-agent-as-ssh
|  |- backup-restic/...
|  |- snapper/...
|- systemd/
|  |- user/waybar.service     # audited user unit(s)
|- render/                    # ephemeral generations (effective files)
|- migrations/                # idempotent, one-time scripts
|  |- 2025-10-02_waybar_unit.sh
|- lib/                       # helpers
|  |- merge.sh                # yq/jq wrappers, JSONC stripper, ini merge
|  |- fs.sh                   # atomic write, symlink ownership
|  |- mods.sh                 # module resolver
|  |- runner.sh               # plan/apply engine
|  |- migrate.sh              # migration runner
|- state/state.json           # applied migrations, checksums, current generation
```

---

## 5) Layering & Merge Semantics

**Layers**
- **L0 core**: project defaults (you own).
- **L1 profile**: curated presets (e.g., laptop vs desktop, GPU variants).
- **L2 host**: per-machine pins (resolution, monitor arrangement, device quirks).
- **L3 user**: personal overrides; **never** modified by the project.

**Merge modes**
- `copy` - one-shot file (we own it).
- `link` - symlink to render directory (preferred when possible).
- `layered` - base file + append `*.d` fragments (e.g., CSS).
- `merge-json` - deep merge JSON/JSONC; user wins on key conflict.
- `merge-yaml` - deep merge YAML; user wins.
- `merge-ini` - per-section per-key overlay; user wins.

**Hyprland**
- Keep `hyprland.conf` **thin**; rely on includes:
  ```ini
  source = ~/.config/hypr/conf.d/*.conf
  source = ~/.config/hypr/overrides.conf   # user file, not owned by project
  ```

**Waybar**
- Generate `~/.config/waybar/config` from:
  - `config.base.json` (project) + `~/.config/waybar/config.d/*.json` (user).
- Start via wrapper `waybar-launch` (merge step + exec) under a systemd user unit.

---

## 6) Module Contract

`modules/<name>/module.yaml`:

```yaml
name: waybar
provides: [bar]            # capabilities this module supplies
requires: []               # optional: capabilities required
conflicts: [hyprpanel]     # optional: incompatible modules
packages: [waybar]         # pacman package names to ensure
env:                       # optional: environment keys -> environment.d
  XDG_CURRENT_DESKTOP: Hyprland
files:                     # list of file operations
  - src: files/config.base.json
    dest: ~/.config/waybar/config
    mode: merge-json       # or copy/layered/merge-yaml/merge-ini/link
  - src: files/style.css
    dest: ~/.config/waybar/style.css
    mode: layered
user_units:                # systemd --user units to install/enable
  - src: files/systemd/waybar.service
    dest: ~/.config/systemd/user/waybar.service
post:                      # post-apply commands (idempotent)
  - systemctl --user daemon-reload
  - systemctl --user enable --now waybar.service
```

**Resolver rules**
- Exactly one provider per capability enabled (e.g., `bar` -> `waybar` or `hyprpanel`).
- If conflicts are detected, `omctl` fails with an actionable message.
- `packages` are installed (opt-in) via pacman; failures abort plan/apply.

---

## 7) Config Model (User-Facing)

### `config.yaml` (L0 defaults)
```yaml
features:
  compositor: hyprland
  bar: waybar
  firewall: ufw
  editor: vscodium
  devshell: true
  terminals: kitty
  keys_agent: true
  backup_restic: true
  snapper: true

profile: cachyos-laptop

options:
  keyboard_layout: "us"
  lock_on_idle_sec: 600
  hypr_gaps_in: 6
  hypr_gaps_out: 12
  terminal_preferred: "ghostty"   # fallback: kitty
```

### `profiles/*.yaml` (L1), `hosts/*.yaml` (L2), `user/overrides.yaml` (L3)
- Any key present in these overlays the previous layer (deep merge).
- L3 (user) **always** wins.

---

## 8) CLI Design (`omctl`)

**Subcommands**
- `omctl doctor` - verify deps (bash, jq, yq, git, systemd-user), repo sanity.
- `omctl plan` - render into `render/$GEN/...`, compute diffs vs live, **do not apply**.
- `omctl apply` - atomically link/copy rendered files, (re)start impacted user units.
- `omctl update` - `git pull` + run new migrations + `plan->apply` in one go.
- `omctl feature set <capability> <module>` - swap providers (e.g., `bar hyprpanel`).
- `omctl rollback` - revert to previous generation (or snapper snapshot if enabled).

**Flags**
- `--dry-run` (alias of `plan`)
- `--yes` (non-interactive apply)
- `--no-pacman` (skip package ensure step)
- `--json` (machine-readable plan output)

**Exit codes**
- `0` success; `1` fatal error; `2` conflict/validation failure; `3` partial apply (see logs).

---

## 9) Apply / Plan Algorithm

1. **Load config**: merge L0->L1->L2->L3 into an effective config object.
2. **Resolve modules**: build a DAG from `provides/requires/conflicts`; detect conflicts.
3. **Ensure packages** (optional): install missing packages for enabled modules.
4. **Render**:
   - For each module file: perform the declared merge mode into `render/$GEN/...`.
   - Write environment keys into `~/.config/environment.d/*.conf`.
   - Generate wrapper scripts (e.g., `~/.local/bin/waybar-launch`) as needed.
5. **Plan**: diff rendered output vs live (symlink targets or files); print summary.
6. **Apply**:
   - Own files by **symlink** when possible (live -> `render/$GEN/...`).
   - If destination is a non-symlink user file, **never overwrite**: write `*.omarch.new` next to it and report.
   - Record checksums and a monotonic **generation** number in `state/state.json`.
7. **Restart** only user units whose configs changed.
8. **Post**: run module `post` commands (idempotent).

**Rollback**
- If `snapper` enabled: roll back last pre-apply snapshot (document flow).
- If using generations: retarget live symlinks to the previous `render/$GEN-1` and restart affected units.

---

## 10) State & Migrations

**`state/state.json`**
```json
{
  "generation": 7,
  "migrations": {
    "2025-10-02_waybar_unit": 1696222222
  },
  "checksums": {
    "home/user/.config/waybar/config": "sha256:...",
    "home/user/.config/hypr/hypridle.conf": "sha256:..."
  }
}
```

**Migrations**
- File name format: `YYYY-MM-DD_<short-desc>.sh` (or `v0.2.0__rename_key.sh`).
- Idempotent; recorded in `state.json`.
- Purely for **structural** changes (move files, rename keys, unit renames).
- Content rules:
  - Detect pre-conditions; do nothing if already migrated.
  - Avoid touching user files unless a clear, safe transformation is guaranteed.
  - Log all changes.

---

## 11) Security Posture

- **ufw**: default deny incoming; allow outgoing; SSH closed unless explicitly opened.
- **SSH (optional)**: key-only auth; `PasswordAuthentication no`, `PermitRootLogin no` (apply only if user enables module).
- **Agent**: `gpg-agent` with `enable-ssh-support` or `keychain`; one keychain UX.
- **AppArmor (later milestone)**: enable + basic profiles; low blast radius.
- **Principle of least privilege**:
  - `omctl` runs as the user except for explicit pacman/ufw operations that require `sudo`.
  - User-unit services (Waybar, hypridle, etc.) live in the user session, not as root.

---

## 12) Observability & Debugging

- CLI logs to stderr with clear prefixes and timestamps.
- `journalctl --user -u <unit>.service` for unit logs.
- `omctl plan --json` emits machine-readable changes; helpful for CI.
- `omctl doctor` prints versions and key environment info.

---

## 13) Testing Strategy

- **Static**: `shellcheck`, `shfmt` for scripts.
- **Unit** (bash helpers): small tests for `merge.sh` (json/yaml strip-and-merge).
- **Integration (VM)**:
  - Fresh CachyOS VM: `omctl plan/apply`, verify Hyprland session, Waybar, portal screenshare.
  - Update path: simulate `git pull` + migration; assert idempotency and no data loss.
- **Matrix**:
  - GPU: AMD/Intel primary; optional NVIDIA profile later.
  - Shells: zsh/fish/bash - ensure shell changes don’t affect startup.

---

## 14) Risks & Mitigations

- **User file clobbering** -> Never overwrite non-owned files; write `*.omarch.new` and show diffs.
- **Conflicting modules** -> Strict resolver with actionable errors.
- **Portal regressions** -> Environment set via `environment.d`; document troubleshooting.
- **Waybar JSON drift** -> JSONC merge wrapper; users place fragments in `config.d/`.

---

## 15) Roadmap Overview

- v0.1.0: core engine (plan/apply), hyprland/waybar/ufw/vscodium/devshell, greetd, portal env, restic, snapper.
- v0.2.0: keys-agent module, terminals module (ghostty/kitty), basic migrations, rollback helper.
- v0.3.0: HyprPanel alternative bar; NVIDIA profile; AppArmor baseline.
- v1.0.0: CI images, docs polished, upgrade guarantees documented.
