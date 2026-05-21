# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A Fedora Server → minimal desktop bootstrapper. The deliverable is a single Bash script (`install.sh`) that installs window-manager / desktop packages via `dnf` and copies a bundled set of dotfiles into the invoking user's home. It is not a library, has no test suite, and no build step.

## Common commands

```bash
./install.sh -h                # usage
./install.sh -V                # version
./install.sh all               # install everything + all configs + fonts + bg + enable sddm
./install.sh essential         # WM stack only (sddm, bspwm, sxhkd, kitty, rofi, polybar, picom, thunar, nitrogen, lxpolkit, ocs-url, fonts, X cfgs)
./install.sh optional          # mangohud, gimp, vim, lxappearance
./install.sh -wc               # install bspwm and copy its config (granular switch form)
./install.sh -v <args>         # verbose / debug logging to stderr
```

The script `sudo`s `dnf`, so it expects an interactive user with sudo rights on Fedora. There is no dry-run.

## Architecture of install.sh

The script is a single file with `parse_args` → `main` flow. Understanding the arg pipeline matters because the meta-args silently override switches:

1. **`parse_args`** uses `getopts` with optstring `:hvVabcfgklmMnopPrsStuwxX`. Each switch appends a package name to `INSTALL_LIST` or sets a flag (`BG`, `CONFIGS`, `FONTS`, `UPDATE`, `SERVICES`, `XCONFIGS`). `-v` and `-c` decrement `NUM_OPTS` so they don't count as "actionable" — this is what makes "only `-v`" fall through to the `all` default.
2. **Meta-arg resolution**: positional `all` / `essential` / `optional` rebuild `INSTALL_LIST` from scratch and force the relevant flags on. If no positional arg and `NUM_OPTS == 0`, it defaults to `all`.
3. **Config matching** (when `-c` / `CONFIGS=y`): for each installed package, iterates `dotconfig/` and selects any entry whose name (first `.`-separated token, uppercased) appears as a substring of the package name uppercased. This is a fuzzy match — adding a new package whose name collides with an existing config dir will pick it up unintentionally.
4. **X configs** (when `-X` / `XCONFIGS=y`): copies every top-level `.X*` and `.x*` file (e.g. `.Xresources`, `.Xnord`, `.xinitrc`) into `$HOME`.
5. **Fonts** (when `-f`): downloads FiraCode + Meslo Nerd Fonts v2.1.0 zips via `wget` to the cwd, unzips into `~/.fonts` (with `-n` to skip existing), runs `fc-cache -fv`. Zips are left in place; `.gitignore` keeps them out of git.
6. **Services** (when `-S` or via `all`/`essential`): `systemctl enable sddm` and `set-default graphical.target`.

The `main` block at the bottom guards execution with `if [[ $(basename $0) == "install.sh" ]]`, so the file can be sourced for testing without running.

## Repo layout

- `install.sh` — the entire installer.
- `dotconfig/` — mirrors `~/.config/`. Subdir names must match (case-insensitively, as substring) the package name they go with, or the matcher in `parse_args` won't pick them up.
- `.Xresources`, `.Xnord`, `.xinitrc` — copied to `$HOME` by the `-X` path.
- `bg.jpg` — copied to `~/.config/bg.jpg` by the `-b` path; bspwmrc references `~/Pictures/background.jpg` (mismatch — nitrogen `--set-auto` in bspwmrc uses the correct `~/.config/bg.jpg`).
- `rpm-packages/ocs-url-3.1.0-1.fc20.x86_64.rpm` — installed by `-o`; path is hard-coded.

## Gotchas when editing

- **Bare `${VAR}` truthiness checks** (e.g. `if [[ ${CONFIGS} ]]`) treat any non-empty string as true. The code sets these to `y`; don't introduce assignments to `n` or `0` expecting them to be false.
- **`INSTALL_LIST[@]` is expanded unquoted** into `dnf -y install`. Don't put spaces in package entries — note the `for font in "fontawesome-fonts fontawesome-fonts-web"` loop, which currently appends one string but `dnf` splits it back apart on word-split.
- **Meta-args wipe granular switches.** If a user runs `./install.sh -g all`, the `-g` is discarded because the `all` branch reassigns `INSTALL_LIST`.
- **`which` truthiness check is inverted-looking but works**: `if [[ ! $(which wget 2>&1>/dev/null) ]]` always evaluates the captured stdout (redirected to `/dev/null` first, so the test is on empty) — effectively always true. Adding `wget`/`unzip` to the install list is therefore unconditional. If you "fix" this, verify behavior across all entry paths.
- **Config-match is substring, case-insensitive, on the package name.** Renaming a `dotconfig/` subdir or adding a new package can cause unintended config copies (or silently skip an intended one).
