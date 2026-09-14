# AeroSpace + JankyBorders

A macOS tiling-window setup: [AeroSpace](https://github.com/nikitabobko/AeroSpace) as the
window manager, [JankyBorders](https://github.com/FelixKratz/JankyBorders) drawing a border
around the focused window, and the border color acting as a **binding-mode indicator** —
amber in service mode, cyan in main mode.

Hand this file to Claude Code on a new Mac and say *"follow the README."*

Verified against: macOS 26.6.2, Apple Silicon, AeroSpace 0.21.3-Beta, JankyBorders 1.9.0.

---

## What you get

| Piece | Role |
|---|---|
| **AeroSpace** | i3-like tiling window manager. Config: `~/.aerospace.toml` |
| **JankyBorders** (`borders`) | Border on the focused window. Runs as a persistent `brew services` daemon |
| **`on-mode-changed` hook** | Repaints the border whenever AeroSpace changes binding mode |

**Service mode is sticky.** `alt+shift+;` enters it, the `join-with` keys can be pressed
repeatedly without dropping out, and `esc` leaves. The border stays amber throughout.

---

## Repo layout

| File | Linked to | Purpose |
|---|---|---|
| `.aerospace.toml` | `~/.aerospace.toml` | The working AeroSpace config — sticky service mode + mode indicator |
| `bordersrc` | `~/.config/borders/bordersrc` | JankyBorders startup defaults (width, style, colors) |
| `record-mode` | — | Script: pillarbox the tiling area to a 16:9 box for screen recording |
| `README.md` | — | This guide |
| `default-config.toml` | — | Pristine stock AeroSpace config, kept for reference only. **Not** the one to link — diff against it to see what this setup changes |

The two linked files are symlinked out of this repo, so you edit them at their normal
`~` paths and git sees the changes.

---

## Setting up a new machine

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

Confirm the binary locations — they are baked into the config:

```bash
brew --prefix                 # /opt/homebrew on Apple Silicon, /usr/local on Intel
which aerospace borders
```

### 3. Clone and link

Do this **before** starting the borders service, so `bordersrc` exists when borders
first launches.

```bash
git clone https://github.com/rileyrichter/aerospace.git ~/code/aerospace-setup
mkdir -p ~/.config/borders

# Back up anything already there
[ -e ~/.aerospace.toml ] && [ ! -L ~/.aerospace.toml ] && mv ~/.aerospace.toml ~/.aerospace.toml.bak

ln -sf ~/code/aerospace-setup/.aerospace.toml ~/.aerospace.toml
ln -sf ~/code/aerospace-setup/bordersrc       ~/.config/borders/bordersrc
chmod +x ~/code/aerospace-setup/bordersrc
```

**Intel Macs only.** The config hardcodes Apple-Silicon paths (`/opt/homebrew/bin/...`).
Rewrite them in the repo file — not through the symlink, which `sed -i` would clobber:

```bash
if [ "$(brew --prefix)" != /opt/homebrew ]; then
  sed -i '' "s|/opt/homebrew/bin/|$(brew --prefix)/bin/|g" ~/code/aerospace-setup/.aerospace.toml
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
`borders` manually from a shell — the service owns the process.

The service starts `borders` with **no arguments**. Per `man borders`, an argument-less
start executes `~/.config/borders/bordersrc` — which step 3 just linked, so the border
comes up with the right width, style, and cyan main-mode color immediately. If you edit
`bordersrc` later, apply it with `brew services restart borders`.

### 5. Grant Accessibility permission — human step

**Claude Code cannot do this.** macOS TCC permissions cannot be granted from the command
line. Ask the user to do it:

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
sed -i '' "s|^start-at-login = false|start-at-login = true|" ~/code/aerospace-setup/.aerospace.toml
aerospace reload-config
```

---

## Verification

Scriptable:

```bash
aerospace reload-config && echo "reload OK"
aerospace config --config-path                        # ~/.aerospace.toml
brew services list | grep borders                     # started
pgrep -lf borders                                     # one process
ls -l ~/.aerospace.toml ~/.config/borders/bordersrc   # both symlinks into the repo

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

One line in `.aerospace.toml` does all of it:

```toml
on-mode-changed = ['exec-and-forget /opt/homebrew/bin/borders active_color=$([ "$(/opt/homebrew/bin/aerospace list-modes --current)" = service ] && echo 0xffE3A63E || echo 0xff5EC8E0)']
```

Design notes worth keeping, because each was a wrong turn first:

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
  terminal. One central callback covers every path, including bindings added later.
- **Repeated `borders` calls are cheap.** Per `man borders`, "if an instance of borders is
  already running, subsequent invocations will update the existing process." No process churn.

### Colors

| Mode | Color | Hex |
|---|---|---|
| main | cyan | `0xff5EC8E0` |
| service | amber | `0xffE3A63E` |

Format is `0xAARRGGBB` (alpha first). To change them, edit the two literals in
`on-mode-changed`, plus `active_color` in `bordersrc` for the startup color — keep the two
cyan values in sync.

### Service-mode bindings

```toml
[mode.service.binding]
    esc = ['reload-config', 'mode main']
    r = ['flatten-workspace-tree', 'mode main']
    f = ['layout floating tiling', 'mode main']
    backspace = ['close-all-windows-but-current', 'mode main']
    c = ['exec-and-forget $HOME/code/aerospace-setup/record-mode toggle', 'mode main']

    alt-shift-h = 'join-with left'
    alt-shift-j = 'join-with down'
    alt-shift-k = 'join-with up'
    alt-shift-l = 'join-with right'
```

The `join-with` bindings deliberately have **no** `'mode main'`. With it, one keypress
silently returned you to main mode, and the next directional press hit main-mode
`move left/down/up/right` — flinging the window to the workspace edge instead of joining it
with a neighbor.

---

## Pinning apps to workspaces

Not in the config yet — this is the one idea worth taking from
[Josean's AeroSpace guide](https://www.josean.com/posts/how-to-setup-aerospace-tiling-window-manager).
Deliberately **not** taking his gaps (this setup runs zero-gap) or his trimmed workspace list.

`on-window-detected` fires once per new window and can route it to a fixed workspace, so an
app always opens where you expect it:

```toml
[[on-window-detected]]
if.app-id = 'com.apple.Safari'
run = 'move-node-to-workspace B'

[[on-window-detected]]
if.app-id = 'com.googlecode.iterm2'
run = 'move-node-to-workspace T'
```

Get the real bundle IDs from the apps you actually run — don't guess them:

```bash
aerospace list-apps        # middle column is the bundle ID
```

Notes:

- These are TOML **array-of-tables** (`[[...]]`), one block per rule. Append them at the
  **end** of the file: a `[[table]]` header is an absolute path, so it stays top-level no
  matter which `[section]` precedes it — but every key after it belongs to it, so dropping
  one into the middle of `[mode.main.binding]` orphans the rest of that section.
- First matching rule wins; rules are checked in file order.
- Only applies to windows detected *after* the rule loads. `aerospace reload-config` does
  not retroactively move windows already open.
- The target workspace should be in `persistent-workspaces` (it already lists 1-9 and A-Z)
  so it survives being emptied.
- Other predicates exist — `if.app-name-regex-substring`, `if.window-title-regex-substring`,
  `if.during-aerospace-startup` — see
  [the guide](https://nikitabobko.github.io/AeroSpace/guide#on-window-detected-callback).

---

## Recording box (16:9)

For Looms and Zoom shares. `alt+shift+;` then **`c`** toggles a centered 16:9 box: the outer
gaps grow until the tiling area *is* the box. One window fills it; open a second and they
tile — inside the box, not across the whole display. `c` again restores full width.

```bash
record-mode                 # toggle, 1920x1080
record-mode on 2496x1404    # explicit size
record-mode off
```

Default is **1920x1080** — captured 1:1 with no rescaling, which is the sharpest a Loom
gets. `2496x1404` is the largest exact 16:9 that fits this display and still downscales to
1080p by a clean 1.3x if you want more room.

Sizes are computed from `NSScreen.visibleFrame`, not the raw resolution, so the menu bar is
accounted for. On the C49RG9 (5120x1440, 30pt menu bar → 5120x1410 usable) a 1080p box
lands at side gaps 1600, top/bottom 165.

Why gaps and not window geometry: AeroSpace has no absolute move/resize — `move` and
`resize` are relative, and there is no "set this window to 1920x1080 at x,y". Gaps are the
only lever that produces a fixed rectangle, and they get tiling inside the box for free.

Two things to know:

- The script rewrites `gaps.outer.*` in `.aerospace.toml` and reloads, so **the repo is
  dirty while the box is on.** Toggle off before committing, or `git checkout .aerospace.toml`.
- Gaps are global — every workspace is boxed, and on a multi-monitor setup the size is
  computed from the main display only.

---

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| No border at all | `brew services list \| grep borders`. If not `started`, `brew services start borders`. JankyBorders needs macOS 14+ |
| Border never changes color | `aerospace reload-config` and check for errors. Confirm the paths in `on-mode-changed` match `which borders` / `which aerospace` — likely an Intel/Apple-Silicon prefix mismatch |
| Border stuck amber | You are still in service mode. `aerospace list-modes --current` to confirm, `aerospace mode main` to escape |
| Windows don't tile | Accessibility permission not granted — see step 5. Quit and relaunch AeroSpace after enabling |
| Wrong color / width at boot | `bordersrc` missing, not executable, or not symlinked. `chmod +x` it, then `brew services restart borders` |
| Config edits don't stick | A symlink was replaced by a regular file. Check `ls -l`; re-create with `ln -sf` |
| Linked the wrong config | `default-config.toml` is the stock file with no fixes. `~/.aerospace.toml` must point at `.aerospace.toml` |
| `borders not found` after install | Shell PATH is stale. Open a new shell, or `eval "$(brew shellenv)"` |
| Stuck in the 16:9 box | `record-mode off`. If the script is gone, set the four `gaps.outer.*` back to `0` and `aerospace reload-config` |
| `record-mode` does nothing | It must be executable and at `~/code/aerospace-setup/record-mode` — that path is baked into the `c` binding |

## References

- AeroSpace — [repo](https://github.com/nikitabobko/AeroSpace) · [commands](https://nikitabobko.github.io/AeroSpace/commands) · [guide](https://nikitabobko.github.io/AeroSpace/guide)
- JankyBorders — [repo](https://github.com/FelixKratz/JankyBorders) · `man borders`
- [Josean's AeroSpace guide](https://www.josean.com/posts/how-to-setup-aerospace-tiling-window-manager) — source of the app-pinning idea; its gaps and workspace trimming are not used here
