# dotfiles

Declarative machine setup for one MacBook and several Windows PCs, driven by [chezmoi](https://www.chezmoi.io).
The only thing you edit is `.chezmoidata/apps.yaml`. chezmoi renders it into a `Brewfile` (macOS) or a
`winget.json` (Windows) plus a `manual.md` checklist for this machine, and runs the package manager when the
rendered list changes. Shell and tool configs can join the same repo later.

## Layout

```
.chezmoidata/apps.yaml                     apps (install method per OS), profiles, machines  <- the source of truth
.chezmoi.toml.tmpl                         asks for the machine id once at init; sets the PowerShell interpreter on Windows
.chezmoiignore                             skips Brewfile on Windows and winget.json on macOS
.chezmoitemplates/app-ids                  resolves machine -> profiles -> app ids, fails loudly on typos
.chezmoitemplates/brewfile, winget-json    render the bundles (also hashed into the install scripts)
dot_config/machine-setup/Brewfile.tmpl     -> ~/.config/machine-setup/Brewfile     (macOS)
dot_config/machine-setup/winget.json.tmpl  -> ~/.config/machine-setup/winget.json  (Windows)
dot_config/machine-setup/manual.md.tmpl    -> ~/.config/machine-setup/manual.md    (apps with no package on this OS)
run_onchange_after_20-install-apps.sh.tmpl   brew bundle --no-upgrade        (macOS only)
run_onchange_after_20-install-apps.ps1.tmpl  winget import --no-upgrade      (Windows only)
```

## New machine

1. Install chezmoi with the OS package manager (no pipe-to-shell one-liners; some endpoint protection flags them).
   - macOS: install [Homebrew](https://brew.sh), then `brew install chezmoi`
   - Windows: `winget install twpayne.chezmoi` (winget ships with Windows 11)
2. Initialise from this repo. The repo is named `dotfiles`, so the GitHub user name is enough.

```
chezmoi init --apply clunghues
```

3. At the prompt, enter the machine id: one of the keys under `machines:` in `apps.yaml` (`pc`, `macbook`, ...).
   It is stored in `~/.config/chezmoi/chezmoi.toml`. Unknown ids stop the apply with the list of valid ones.
4. Apps install through `brew bundle` or `winget import`. Already installed apps are skipped. Some installers may
   ask for a password (macOS) or show a UAC prompt (Windows). Then open `~/.config/machine-setup/manual.md`.

Preview without changing anything: `chezmoi apply --dry-run --verbose`.

## Daily use

```
chezmoi update                 # pull + apply on this machine (installs apps added since last time)
chezmoi diff                   # what apply would change (files and scripts; not missing apps)
chezmoi edit-config            # change this machine's id / profiles selection
chezmoi cd                     # open a shell in the source checkout to edit apps.yaml and commit
```

Add an app: put it under `apps:` with its install method per OS, list its id in a profile, commit and push.
Other machines pick it up on their next `chezmoi update`.

```yaml
apps:
  obsidian:
    name: Obsidian
    darwin:  { provider: cask,   package: obsidian }
    windows: { provider: winget, package: Obsidian.Obsidian }
profiles:
  core: [obsidian]
machines:
  pc:      { profiles: [core, home] }
  macbook: { profiles: [core] }
```

Providers: macOS `brew`, `cask`, `mas` (`id:`), `manual` (`url:`); Windows `winget`, `manual`, `scoop`
(scoop entries are listed in `manual.md` for now, not installed). Optional `tap:` on a brew/cask entry adds a tap line.

Seeding the list from an existing machine: `brew bundle dump --file=-` (macOS), `winget export -o x.json` and
`scoop export` (Windows) show what is installed. Pick the apps you want and add them to `apps.yaml` by hand.

## Policies

- Apply installs what is missing. It never runs an explicit upgrade and never uninstalls. Upgrade with
  `brew upgrade` / `winget upgrade --all` yourself. Removing an app from a profile only stops installing it elsewhere.
- No version pinning. Changing a provider (winget -> scoop) does not migrate an existing install.
- The install scripts re-run only when the rendered bundle changes, or after a failed run. If you removed an app by
  hand and want it back, force a re-run:

```
chezmoi state delete-bucket --bucket=scriptState && chezmoi apply
```

- Secrets and machine-specific paths do not belong here. Per-machine values live in `~/.config/chezmoi/chezmoi.toml`.
- Agent configuration (Claude Code / Codex skills) is a separate repo: `npx -y github:clunghues/agent-skills`.
