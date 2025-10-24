# TODO — Omarch‑Alt (CachyOS + Hyprland)

> Use this as the running backlog. Each task has **Acceptance Criteria (AC)**.
> Milestones group tasks into shippable slices.

---

## M0 — Repo Bootstrap

- [ ] Create repo skeleton (see System_Design.md §4).
  - AC: `omctl`, `lib/`, `modules/`, `profiles/`, `user/`, `render/`, `state/` exist.
- [ ] Add `.editorconfig`, `.gitignore`, `LICENSE`, `CODE_OF_CONDUCT.md`.
  - AC: repo passes basic hygiene checks.
- [ ] Add `README.md` with quick start (deps, clone, `omctl plan/apply`).
  - AC: A new user can follow steps and reach the Hyprland session.

---

## M1 — Core Engine (plan/apply)

- [ ] Implement `omctl` CLI skeleton with subcommands: `doctor`, `plan`, `apply`, `update`, `feature`, `rollback`.
  - AC: `omctl help` lists commands; returns non‑zero on unknown command.
- [ ] `lib/merge.sh`: `merge-json`, `merge-yaml` (yq), `strip_jsonc`, basic `merge-ini`.
  - AC: Given test fixtures, merge produces expected outputs.
- [ ] `lib/fs.sh`: atomic writes, symlink ownership, `.omarch.new` fallback.
  - AC: Applying to an existing non‑symlink produces `.omarch.new`, not overwrite.
- [ ] `lib/mods.sh`: module loader/resolver (provides/requires/conflicts).
  - AC: Enabling two conflicting modules fails with actionable message.
- [ ] `lib/runner.sh`: end‑to‑end plan/apply pipeline.
  - AC: Renders to `render/$GEN`; prints diffs; applies changes atomically; restarts changed units only.
- [ ] `omctl doctor`: checks for `bash`, `jq`, `yq`, `git`, systemd user session.
  - AC: Friendly output; failure if a dependency is missing.

---

## M2 — Baseline Modules (MVP Desktop)

### Hyprland module
- [ ] `modules/hyprland/module.yaml` with packages, files, env.
- [ ] Files: `hyprland.conf` (thin includes), `conf.d/00-base.conf`, `hypridle.conf`, `hyprpaper.conf`.
  - AC: Hyprland starts; `hypridle` locks after `options.lock_on_idle_sec`; wallpaper loads.

### Waybar module
- [ ] `modules/waybar/module.yaml` with JSONC base, CSS, systemd user unit.
- [ ] Wrapper `~/.local/bin/waybar-launch` merges `config.base.json` + `config.d/*.json`.
  - AC: Waybar starts via systemd user; custom fragment in `config.d/` overrides base.

### Portal & environment
- [ ] Write `XDG_CURRENT_DESKTOP=Hyprland`, `GDK_BACKEND=wayland`, `QT_QPA_PLATFORM=wayland` into `~/.config/environment.d/10-hyprland.conf`.
  - AC: Screenshare works in Wayland apps (Firefox/OBS).

### Login
- [ ] `greetd + tuigreet` configuration (default session → `Hyprland`).
  - AC: Login manager works; **no shell RC** involved.

### Firewall
- [ ] `modules/ufw/` with default deny incoming, allow outgoing.
  - AC: `ufw status` shows enabled; inbound closed.

### Editor & Devshell
- [ ] `modules/vscodium/` installs VSCodium.
  - AC: `codium` runs.
- [ ] `modules/devshell/` installs zsh, starship, zoxide, fzf, ripgrep, btop (no shell RC dependence).
  - AC: Packages present; user may opt‑in to prompt init separately.

---

## M3 — Safety Nets

- [ ] `modules/backup-restic/`: repo env, timer, exclude file.
  - AC: `systemctl list-timers` shows daily backup; `restic snapshots` shows runs.
- [ ] `modules/snapper/`: create config for `/`, enable timeline + snap‑pac.
  - AC: `snapper list` shows timeline; pacman pre/post snapshots appear.
- [ ] `omctl rollback`: previous generation retarget (symlinks) and/or call out to snapper.
  - AC: After a bad apply, `omctl rollback` restores prior generation.

---

## M4 — Update & Migrations

- [ ] `lib/migrate.sh`: run idempotent scripts in `migrations/`, record in `state/state.json`.
  - AC: Re‑running update doesn’t reapply old migrations.
- [ ] `omctl update`: `git pull --ff-only` + `migrate` + `plan→apply`.
  - AC: Owner adds a new migration; `omctl update` applies it once and updates state.
- [ ] Sample migration: rename Waybar unit or relocate a config path.
  - AC: Migration changes are visible and logged; no user files clobbered.

---

## M5 — Feature Swaps & Profiles

- [ ] Implement `omctl feature set <cap> <module>` to change provider (e.g., `bar hyprpanel` later).
  - AC: Resolver validates conflicts; plan shows removal/addition of units/files.
- [ ] Add `profiles/cachyos-laptop.yaml` (PPD, bluetooth, power tweaks).
  - AC: Selecting this profile changes effective config accordingly.
- [ ] (Optional) Add `modules/hyprpanel/` as an alternative to Waybar.
  - AC: Swapping `features.bar: hyprpanel` works via plan/apply.

---

## M6 — Keys & Terminals

- [ ] `modules/keys-agent/`: generate SSH ed25519, GPG keys; set `gpg-agent` with `enable-ssh-support`.
  - AC: `SSH_AUTH_SOCK` points to gpg‑agent; `ssh -T git@github.com` works.
- [ ] `modules/terminals/`: install ghostty + kitty; `options.terminal_preferred` selects default in Hypr binds.
  - AC: Mod+Enter launches selected terminal.

---

## M7 — Testing & CI

- [ ] Add `shellcheck` + `shfmt` lint job (GitHub Actions).
  - AC: PRs fail on lint errors.
- [ ] Smoke test script for a fresh CachyOS VM (non‑interactive).
  - AC: Script gets to Hyprland + running Waybar with portal enabled.
- [ ] Document manual test matrix (AMD/Intel GPU; Zsh/Fish/Bash shells).
  - AC: Checklist exists and can be followed.

---

## M8 — Documentation

- [ ] Flesh out `System_Design.md` (this doc) with any deviations found during implementation.
- [ ] `docs/USAGE.md`: install deps, run `omctl`, override examples, module swaps, recovery.
- [ ] `docs/RECOVERY.md`: snapper rollback, restic restore, safe mode login (TTY), greetd disable.
- [ ] `docs/CONTRIBUTING.md`: coding style, PR conventions, commit format, versioning.

---

## M9 — v0.1.0 Release

- [ ] Tag v0.1.0; changelog `CHANGELOG.md`.
- [ ] Draft GitHub release with screenshots (Hyprland, Waybar, lock screen).
- [ ] Issue templates: bug report, feature request.
- [ ] Roadmap issues for v0.2.0 (keys/terminals/migrations polish).

---

## Nice‑to‑Haves / Later

- [ ] NVIDIA profile (separate module/profile, optional env toggles).
- [ ] AppArmor baseline and common profiles.
- [ ] Podman (+ Quadlet) module for dev containers; VS Code DevContainers docs.
- [ ] Optional HyprPlugins via `hyprpm` with pinned versions.
- [ ] Autogen monitors config from `hyprctl monitors -j` on first run.

---

## Done Definition (per milestone)

- **Green doctor**; **plan** shows expected diffs; **apply** succeeds; session survives logout/login; user overrides respected; rollback path documented.

