# AeroSpace + JankyBorders — Setup Guide

Reproduces a macOS tiling-window setup on a new machine: AeroSpace as the window
manager, JankyBorders drawing a colored border around the focused window, and the
border color acting as a **binding-mode indicator** (amber = service mode, cyan = main mode).

This file is written to be handed to Claude Code on the new Mac. Part A is a one-time
step on the Mac that already works; Part B is the new-machine install.

Verified against: macOS 26.6.2, Apple Silicon, AeroSpace 0.21.3-Beta, JankyBorders 1.9.0.

---

## What you end up with

| Piece | Role |
|---|---|
| **AeroSpace** | i3-like tiling window manager. Config: `~/.aerospace.toml` |
| **JankyBorders** (`borders`) | Draws a border on the focused window. Runs as a persistent `brew services` daemon |
| **`on-mode-changed` hook** | Repaints the border whenever AeroSpace changes binding mode |

**Service mode is sticky.** `alt+shift+;` enters it, the `join-with` keys can be pressed
repeatedly without dropping out, and `esc` leaves. The border stays amber for the whole
time you are in service mode.

### Files this setup owns

| Path | Purpose |
|---|---|
| `~/.aerospace.toml` | AeroSpace config, including the mode-indicator hook |
| `~/.config/borders/bordersrc` | JankyBorders startup defaults (width, style, colors) |
| `aerospace-setup.md` | This guide |

All three belong in the dotfiles repo.

---

## Part A — one-time, on the Mac that already works

Put the files in a GitHub repo so the new machine can pull them. If you already keep
dotfiles in a repo, use that one and skip `gh repo create`.

```bash
DOTS=~/dotfiles
mkdir -p "$DOTS/.config/borders" && cd "$DOTS"
git init -b main 2>/dev/null || true
```

Move the real files into the repo and leave symlinks behind:

```bash
mv ~/.aerospace.toml              "$DOTS/.aerospace.toml"
mv ~/.config/borders/bordersrc    "$DOTS/.config/borders/bordersrc"
mv ~/aerospace-setup.md           "$DOTS/aerospace-setup.md"

ln -sf "$DOTS/.aerospace.toml"           ~/.aerospace.toml
ln -sf "$DOTS/.config/borders/bordersrc" ~/.config/borders/bordersrc
```

The symlinks matter: you edit the paths in `~` as usual, and git sees the changes.

```bash
git add .aerospace.toml .config/borders/bordersrc aerospace-setup.md
git commit -m "Add AeroSpace + JankyBorders config"
gh repo create dotfiles --private --source=. --push
```

> Verify nothing broke: `ls -l ~/.aerospace.toml ~/.config/borders/bordersrc` should
> both show `-> …/dotfiles/…`, and `aerospace reload-config` should exit 0.

**Note on editing later:** some editors and `sed -i` replace a symlink with a regular
file instead of writing through it. If `ls -l` stops showing the arrow, edit the file in
the repo directly and re-create the symlink.

---

## Part B — new machine

Give Claude Code this file and say: *"follow aerospace-setup.md."*

Set the repo URL first:

```bash
REPO_URL=git@github.com:<YOUR_USER>/dotfiles.git    # <-- fill in
```

### 1. Prerequisites

```bash
# Homebrew — skip if `brew --version` already works
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

xcode-select --install 2>/dev/null || true   # ok if already installed
```

Requirements: macOS 13+ for AeroSpace, macOS 14+ for JankyBorders.

### 2. Install AeroSpace and JankyBorders

```bash
brew install --cask nikitabobko/tap/aerospace
brew install FelixKratz/formulae/borders
```

Confirm the binary locations before going further — they are baked into the config:

```bash
brew --prefix                 # /opt/homebrew on Apple Silicon, /usr/local on Intel
which aerospace borders
```

### 3. Clone the dotfiles and link them into place

Do this **before** starting the borders service, so `bordersrc` exists when borders
first launches.

```bash
git clone "$REPO_URL" ~/dotfiles
mkdir -p ~/.config/borders

# Back up anything already there, then symlink
[ -e ~/.aerospace.toml ] && [ ! -L ~/.aerospace.toml ] && mv ~/.aerospace.toml ~/.aerospace.toml.bak

ln -sf ~/dotfiles/.aerospace.toml           ~/.aerospace.toml
ln -sf ~/dotfiles/.config/borders/bordersrc ~/.config/borders/bordersrc
chmod +x ~/dotfiles/.config/borders/bordersrc
```

**Intel Macs only.** The config hardcodes Apple-Silicon paths (`/opt/homebrew/bin/...`).
On Intel, rewrite them in the repo file — not through the symlink, which `sed -i` would
clobber:

```bash
if [ "$(brew --prefix)" != /opt/homebrew ]; then
  sed -i '' "s|/opt/homebrew/bin/|$(brew --prefix)/bin/|g" ~/dotfiles/.aerospace.toml
fi
```

Don't commit that rewrite back if your other Macs are Apple Silicon.

### 4. Run borders as a persistent service

```bash
brew services start borders
brew services list | grep borders    # expect: started
```

This installs a LaunchAgent at `~/Library/LaunchAgents/sh.brew.borders.plist` with
`RunAtLoad` and `KeepAlive`, so borders survives reboot and logout. Don't launch
`borders` manually from a shell — the service already owns the process.

The service starts `borders` with **no arguments**. Per `man borders`, an argument-less
start executes `~/.config/borders/bordersrc` — which step 3 just put in place, so the
border comes up with the right width, style, and cyan main-mode color immediately. If you
ever edit `bordersrc`, apply it with `brew services restart borders`.

### 5. Grant Accessibility permission — human step

**Claude Code cannot do this.** macOS TCC permissions cannot be granted from the
command line. Ask the user to do it:

1. Launch AeroSpace (`open -a AeroSpace`). macOS shows a permission prompt.
2. **System Settings → Privacy & Security → Accessibility** → enable **AeroSpace**.
3. If AeroSpace was running before the toggle, quit and relaunch it.

Without this, AeroSpace starts but will not move or tile any windows.

### 6. Start and verify

```bash
open -a AeroSpace
aerospace reload-config && echo "config OK"
```

Optional — start AeroSpace at login. The committed config has `start-at-login = false`:

```bash
sed -i '' "s|^start-at-login = false|start-at-login = true|" ~/dotfiles/.aerospace.toml
aerospace reload-config
```

---

## Verification checklist

Scriptable:

```bash
aerospace reload-config && echo "reload OK"
brew services list | grep borders             # started
pgrep -lf borders                             # one process
aerospace list-modes --current                # main
ls -l ~/.aerospace.toml ~/.config/borders/bordersrc   # both symlinks into ~/dotfiles

# Mode switching drives the right color
aerospace mode service; sleep 1; aerospace list-modes --current   # service
aerospace mode main;    sleep 1; aerospace list-modes --current   # main
```

Needs human eyes:

- [ ] `alt+shift+;` → border turns **amber**
- [ ] In service mode, `alt-shift-h/j/k/l` joins repeatedly, staying amber between presses
- [ ] `esc` → back to main, border returns to **cyan**
- [ ] `r` / `f` / `backspace` still flatten / toggle float / close-others, and return to cyan
- [ ] Reboot → borders is running and the border is cyan with no manual step

---

## How the mode indicator works

One line in `~/.aerospace.toml` does all of it:

```toml
on-mode-changed = ['exec-and-forget /opt/homebrew/bin/borders active_color=$([ "$(/opt/homebrew/bin/aerospace list-modes --current)" = service ] && echo 0xffE3A63E || echo 0xff5EC8E0)']
```

Design notes worth keeping, because each one was a wrong turn first:

- **AeroSpace exports no `AEROSPACE_MODE`.** The only env vars the server exports are
  `AEROSPACE_FOCUSED_WORKSPACE`, `AEROSPACE_PREV_WORKSPACE`, `AEROSPACE_WORKSPACE`, and
  `AEROSPACE_WINDOW_ID`. The callback has to ask `aerospace list-modes --current` which
  mode it is in. Don't "simplify" this into a variable that doesn't exist.
- **No race.** The callback observes the *committed* mode, so the color is never inverted.
- **Absolute paths are deliberate.** `exec-and-forget` runs under a minimal PATH; bare
  `borders` or `aerospace` may not resolve.
- **`on-mode-changed`, not per-binding hooks.** An earlier version appended an
  `exec-and-forget` to each of the five mode-switching keys. That leaves the border lying
  whenever the mode changes by any other path — notably `aerospace mode main` from a
  terminal. One central callback covers every path, including service-mode bindings added
  later.
- **Repeated `borders` calls are cheap.** Per `man borders`, "if an instance of borders is
  already running, subsequent invocations will update the existing process." No process churn.

### Colors

| Mode | Color | Hex |
|---|---|---|
| main | cyan | `0xff5EC8E0` |
| service | amber | `0xffE3A63E` |

Format is `0xAARRGGBB` (alpha first). To change them, edit the two literals in
`on-mode-changed`, plus `active_color` in `bordersrc` for the startup color — keep the
two cyan values in sync.

### `bordersrc`

Kept in the repo at `.config/borders/bordersrc`, symlinked to `~/.config/borders/bordersrc`,
and executed by the service at launch. Must stay executable.

```bash
#!/bin/bash
options=(
  style=round
  width=6.0
  hidpi=on
  active_color=0xff5EC8E0     # cyan  — main mode
  inactive_color=0xff494d64
)
borders "${options[@]}"
```

### Service-mode bindings

```toml
[mode.service.binding]
    esc = ['reload-config', 'mode main']
    r = ['flatten-workspace-tree', 'mode main']
    f = ['layout floating tiling', 'mode main']
    backspace = ['close-all-windows-but-current', 'mode main']

    alt-shift-h = 'join-with left'
    alt-shift-j = 'join-with down'
    alt-shift-k = 'join-with up'
    alt-shift-l = 'join-with right'
```

The `join-with` bindings deliberately have **no** `'mode main'`. With it, one keypress
silently returned you to main mode, and the next directional press hit main-mode
`move left/down/up/right` — flinging the window to the workspace edge instead of joining.

---

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| No border at all | `brew services list \| grep borders`. If not `started`, `brew services start borders`. JankyBorders needs macOS 14+ |
| Border never changes color | `aerospace reload-config` and check for errors. Confirm the paths in `on-mode-changed` match `which borders` / `which aerospace` — likely an Intel/Apple-Silicon prefix mismatch |
| Border stuck amber | You are still in service mode. `aerospace list-modes --current` to confirm, `aerospace mode main` to escape |
| Windows don't tile | Accessibility permission not granted — see step 5. Quit and relaunch AeroSpace after enabling |
| Wrong color / wrong width at boot | `~/.config/borders/bordersrc` missing, not executable, or not symlinked. `chmod +x` it, then `brew services restart borders` |
| Config edits don't stick | A symlink was replaced by a regular file. Check `ls -l`; re-create with `ln -sf` |
| `borders not found` after install | Shell PATH is stale. Open a new shell, or `eval "$(brew shellenv)"` |

## References

- AeroSpace — https://github.com/nikitabobko/AeroSpace ([commands](https://nikitabobko.github.io/AeroSpace/commands), [guide](https://nikitabobko.github.io/AeroSpace/guide))
- JankyBorders — https://github.com/FelixKratz/JankyBorders (`man borders`)
