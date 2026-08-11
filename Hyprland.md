# Hyprland + Celestia Shell 🪟🪽

Installing Hyprland (scrolling-tiling Wayland compositor with native **Scrolling Layout**) with the Celestia shell on an existing Arch Linux install. Celestia is a Quickshell-based desktop shell — bar, launcher, notifications, lock screen, idle, wallpaper, and color picker all in one.

> **This guide targets Hyprland 0.55+.** Since 0.55, Hyprland's config language is **Lua** (`hyprland.lua`) — the old hyprlang syntax (`hyprland.conf`) is deprecated. Check your version with `hyprctl version` before following any config examples.
>
> **Sources:** [Hyprland wiki](https://wiki.hypr.land/) (crawled 2026-08-11), [Celestia shell repo](https://github.com/caelestia-dots/shell), [SDDM](https://github.com/sddm/sddm).

---

## 📖 Table of Contents

1. [Overview](#overview)
2. [Package Stack — Lean But Feature-Rich](#package-stack--lean-but-feature-rich)
3. [Installation](#installation)
4. [Configuration (Lua)](#configuration-lua)
5. [ScrollOverview — Niri-Style Overview](#scrolloverview--niri-style-overview)
6. [Celestia Shell Setup](#celestia-shell-setup)
7. [SDDM Login Manager](#sddm-login-manager)
8. [XWayland](#xwayland)
9. [PCoIP Keyboard Passthrough](#pcoip-keyboard-passthrough)
10. [Troubleshooting](#troubleshooting)
11. [Uninstalling](#uninstalling)

---

## Overview

| Component | Package | Source | Purpose |
|-----------|---------|--------|---------|
| **Hyprland** | `hyprland` | `extra` (official) | Wayland compositor with native scrolling layout |
| **ScrollOverview** | `hyprland-scroll-overview` | plugin (hyprpm) | Niri-style overview: zoom out all workspaces, swipe gesture, ALT+Tab switcher |
| **Celestia shell** | `caelestia-shell` | AUR | Desktop shell: bar, launcher, notifications, lock, idle, wallpaper, picker, dashboard, OSD |
| **SDDM** | `sddm` | `extra` (official) | Login manager (≥ 0.20.0 to avoid bug #1476) |
| **XDG Desktop Portal** | `xdg-desktop-portal-hyprland` | `extra` | Screen sharing / portal integration |
| **Polkit agent** | `hyprpolkitagent` | `extra` | GUI authentication dialogs |
| **XWayland** | `xorg-xwayland` | pulled by Hyprland | Legacy X11 app support (needed — keep enabled) |

**Why this stack:**
- **Celestia replaces 6 separate tools** — no `hyprlock` / `hypridle` / `hyprpaper` / `hyprpicker` / `waybar` / `mako` needed. One Quickshell-based shell covers bar, launcher, notifications, lock screen, idle, wallpaper, color picker, dashboard, and OSD.
- **Scrolling layout is native** in Hyprland 0.55+ — the Niri-style infinitely-growing tape paradigm, no `hyprscroller` plugin.
- **ScrollOverview plugin adds the Niri-style overview** — zoom out to see all workspaces, trackpad swipe, and a visual ALT+Tab switcher (Hyprland has no built-in overview).
- **No `hyprsunset`** — blue light filtering not needed on this setup.

**Key differences from Niri + Noctalia:**
- Config is **Lua** (`~/.config/hypr/hyprland.lua`), not KDL/TOML
- Scrolling layout is **built into the compositor**, not a shell feature
- Celestia is **Quickshell/QML** (like Noctalia v4, not the v5 C++ rewrite)

---

## Package Stack — Lean But Feature-Rich

```
hyprland                      # compositor (official tagged release — NOT -git)
hyprland-scroll-overview      # overview plugin (via hyprpm — Niri-style overview)
caelestia-shell               # shell (AUR — bar/launcher/notif/lock/idle/paper/picker)
grim slurp satty              # screenshots: capture + region select + native annotation
wl-clipboard                  # Wayland clipboard (screenshot copy)
xdg-desktop-portal-hyprland   # screen share + portal
hyprpolkitagent               # polkit auth dialogs
sddm                          # login manager (>= 0.20.0)
```

**Deliberately NOT installed:** `hyprlock`, `hypridle`, `hyprpaper`, `hyprpicker`, `hyprsunset`, `waybar`, `mako`, `rofi`, `fuzzel` — all provided by Celestia.

> **Why official over AUR/git for Hyprland:** the wiki is explicit — *"heavily recommended you use what the distro packages for you, and not compiling manually or using `-git` packages."* Hyprland's ecosystem and dependencies are vast and intertwined; `-git` builds break with `.so` mismatches on every ABI-breaking dependency update (`hyprutils`, etc.).

---

## Installation

### 1. Install the compositor (official)

```bash
sudo pacman -S hyprland
```

This pulls `xorg-xwayland` (XWayland support), `aquamarine`, `hyprutils`, `hyprlang`, etc. automatically.

### 2. Install the ecosystem packages

```bash
sudo pacman -S xdg-desktop-portal-hyprland hyprpolkitagent sddm
```

> **Why:** `xdg-desktop-portal-hyprland` (XDPH) enables screen sharing under Wayland (also required for DBus global shortcuts). `hyprpolkitagent` shows GUI auth dialogs (e.g. package managers). `sddm` is the login manager.

### 3. Install ScrollOverview plugin (Niri-style overview)

```bash
# Add the plugin repository and build it
hyprpm add https://github.com/yayuuu/hyprland-scroll-overview.git
hyprpm update
hyprpm enable scrolloverview
```

> **Why:** Hyprland has no built-in overview (unlike Niri's `Mod+R`). ScrollOverview adds the Niri-style zoomed-out overview of all workspaces, a trackpad swipe gesture, and a visual ALT+Tab switcher. `hyprpm` is Hyprland's bundled plugin manager. If you run a git build of Hyprland, add the `new-release` branch instead: `hyprpm add https://github.com/yayuuu/hyprland-scroll-overview.git origin/new-release`.

### 4. Install screenshot stack (native)

```bash
sudo pacman -S grim slurp satty wl-clipboard
```

> **Why:** `grim` captures the screen (Wayland-native, freedesktop), `slurp` selects a region, `satty` annotates (native GTK/Adwaita, OpenGL-accelerated, wlroots — Hyprland wiki's recommended annotation tool), `wl-clipboard` copies to the Wayland clipboard. No portal needed — this is the lean, native stack. (Flameshot works but is X11-first and relies on portal support on Wayland.)

### 5. Install Celestia shell (AUR)

```bash
yay -S caelestia-shell
```

> **AUR policy — review the PKGBUILD first** (following the [June 2026 AUR malware incident](https://archlinux.org/news/active-aur-malicious-packages-incident/) checklist): `yay -S caelestia-shell --editmenu` or inspect via `yay -G caelestia-shell` before building. Celestia builds against `quickshell-git` (the git version is required, not the tagged release) and pulls `ddcutil`, `brightnessctl`, `libcava`, `fish`, `lm-sensors`, `swappy`, `libqalculate`, `material-symbols` fonts, and a Nerd Font. These are real build/runtime deps — expect a longer install.
>
> Prefer the **stable** package `caelestia-shell` over `caelestia-shell-git` (bleeding edge, unstable).

### 6. Enable SDDM

```bash
sudo systemctl enable sddm --now
```

> **Why ≥ 0.20.0:** SDDM bug [#1476](https://github.com/sddm/sddm/issues/1476) causes 90-second shutdowns. Arch's `sddm` package is current (0.21+), so this is already satisfied — just don't downgrade.

### 7. Verify

```bash
hyprctl version
# expect: Hyprland X.Y.Z (tagged release, not git)
```

---

## Configuration (Lua)

Hyprland 0.55+ reads `~/.config/hypr/hyprland.lua` (not `.conf`). It hot-reloads **the moment you save**. Split config across files with `require()` — each file is a separate Lua scope, so an error in one file doesn't kill the others.

```
~/.config/hypr/
├── hyprland.lua          # main: hl.config + require all
├── keybinds.lua          # hl.bind() — scrolling binds included
├── rules.lua             # hl.window_rule + hl.workspace_rule
├── animations.lua        # hl.animation + hl.gesture
└── celestia.lua          # shell autostart + global shortcut binds
```

### 4.1 Main config (`hyprland.lua`)

```lua
-- ~/.config/hypr/hyprland.lua
local mainMod = "SUPER"   -- the main mod key, used everywhere

require("keybinds")
require("rules")
require("animations")
require("celestia")

hl.config({
  general = {
    border_size = 2,
    gaps_in = 5,
    gaps_out = 10,
    layout = "scrolling",          -- native scrolling layout as the default
    col.active_border = "0xff89b4fa",
    col.inactive_border = "0xff45475a",
  },
  decoration = {
    rounding = 10,
    blur = { enabled = true, size = 8, passes = 3 },
    shadow = { enabled = true, range = 20 },
  },
  animations = {
    enabled = true,
    bezier = { "myBezier, 0.05, 0.9, 0.1, 1.0" },
    animation = {
      "windows, 1, 6, myBezier",
      "fade, 1, 6, default",
      "workspaces, 1, 6, myBezier",
    },
  },
  input = {
    kb_layout = "us",
    follow_mouse = 1,
    touchpad = { natural_scroll = true },
  },
})

-- Scrolling layout (native, 0.55+)
hl.config({
  scrolling = {
    direction = "right",           -- new windows appear + scroll to the right
    column_width = 0.5,            -- default column width [0.1-1.0]
    follow_focus = true,           -- auto-scroll to keep focused window visible
    follow_min_visible = 0.4,      -- min visible fraction before following
    wrap_focus = true,             -- focus l/r wraps at the ends
    wrap_swapcol = true,
    fullscreen_on_one_column = true,
  },
})
```

> **Scrolling layout explained:** windows live on an infinitely growing tape (Niri-style). `direction` controls where new windows appear and which way the layout scrolls. `follow_focus` keeps the focused window in view automatically — hard input (binds/clicks) always follows, `follow_min_visible` only applies to soft input.

### 4.2 Keybinds (`keybinds.lua`)

```lua
-- ~/.config/hypr/keybinds.lua
local mainMod = "SUPER"

--  Launch
hl.bind(mainMod .. " + Return", hl.dsp.exec_cmd("ghostty"))       -- terminal
hl.bind(mainMod .. " + Space", hl.dsp.exec_cmd("caelestia shell launch"))  -- launcher
hl.bind(mainMod .. " + B", hl.dsp.exec_cmd("ghostty -e nvim"))     -- editor

--  Window management
hl.bind(mainMod .. " + Q", hl.dsp.killactive())                     -- close window
hl.bind(mainMod .. " + V", hl.dsp.window.float({ action = "toggle" }))
hl.bind(mainMod .. " + F", hl.dsp.window.fullscreen({ action = "toggle", layout_aware = true }))

--  Scrolling layout: navigate columns
hl.bind(mainMod .. " + H", hl.dsp.layout("focus l"))
hl.bind(mainMod .. " + L", hl.dsp.layout("focus r"))
hl.bind(mainMod .. " + period", hl.dsp.layout("move +col"))         -- scroll right by column
hl.bind(mainMod .. " + comma", hl.dsp.layout("move -col"))          -- scroll left by column

--  Scrolling layout: columns
hl.bind(mainMod .. " + Ctrl + H", hl.dsp.layout("swapcol l"))       -- swap column left
hl.bind(mainMod .. " + Ctrl + L", hl.dsp.layout("swapcol r"))       -- swap column right
hl.bind(mainMod .. " + R", hl.dsp.layout("colresize +conf"))        -- cycle width presets
hl.bind(mainMod .. " + Shift + F", hl.dsp.layout("fit active"))     -- fit active column
hl.bind(mainMod .. " + Shift + C", hl.dsp.layout("fit_into_view"))  -- fit active into view

--  Workspaces
hl.bind(mainMod .. " + Tab", function()
  hl.plugin.scrolloverview.overview("toggle all")   -- Niri-style overview
end)
for i = 1, 9 do
  hl.bind(mainMod .. " + " .. i, hl.workspace(i))
  hl.bind(mainMod .. " + SHIFT + " .. i, hl.dsp.movetoworkspace(i))
end
hl.bind(mainMod .. " + E", hl.dsp.workspace("e+1"))                -- next empty workspace
hl.bind(mainMod .. " + SHIFT + Tab", hl.dsp.workspace("e-1"))      -- prev workspace

--  Screenshots (grim + slurp + satty — native)
hl.bind("Print", hl.dsp.exec_cmd('grim -g "$(slurp)" - | satty -f -'))              -- region → annotate
hl.bind(mainMod .. " + Print", hl.dsp.exec_cmd('grim - | satty -f -'))               -- fullscreen → annotate
hl.bind("CTRL + Print", hl.dsp.exec_cmd('grim -g "$(slurp -d)" - | wl-copy'))         -- region → clipboard

--  System
hl.bind(mainMod .. " + L", hl.dsp.exec_cmd("caelestia shell lock lock"))   -- lock (Celestia)
hl.bind(mainMod .. " + X", hl.dsp.exec_cmd("shutdown now"))                -- power off
```

> **Why `layout_aware = true` on fullscreen:** on scrolling workspaces this lets you **scroll away from a fullscreen window without unfullscreening it** (layout-handled fullscreen).
>
> **Keybind callbacks must not block** — the compositor event loop runs them. Use `hl.dsp.exec_cmd(...)` for anything external; don't call `wl-paste`/`io.popen`/network I/O inside a bind function (a hung bind freezes the whole desktop).

### 4.3 Rules (`rules.lua`)

```lua
-- ~/.config/hypr/rules.lua
-- Per-app scrolling column width
hl.window_rule({ name = "kitty_starting_width", match = { class = "kitty" }, scrolling_width = 0.5 })

-- Float certain dialogs
hl.window_rule({ match = { class = "pavucontrol" }, float = true })
hl.window_rule({ match = { title = "^(Open File)" }, float = true })

-- Per-workspace scrolling direction (e.g. workspace 2 grows downward)
hl.workspace_rule({ workspace = "2", layout_opts = { direction = "down" } })

-- Monitor setup (adjust to your hardware)
hl.monitor("DP-1, 2560x1440@144, 0x0, 1")
hl.monitor("eDP-1, 1920x1080@60, 2560x0, 1")
```

> **Window rules match by** `class`, `title`, `name`, or more (see wiki). Monitor names are `DP-*`, `eDP-*`, `HDMI-A-*` — run `hyprctl monitors` to list yours.

### 4.4 Animations & gestures (`animations.lua`)

```lua
-- ~/.config/hypr/animations.lua
hl.config({
  animations = {
    enabled = true,
    bezier = { "myBezier, 0.05, 0.9, 0.1, 1.0", "linear, 0, 0, 1, 1" },
    animation = {
      "windows, 1, 6, myBezier",
      "windowsOut, 1, 6, default, popin 80%",
      "fade, 1, 6, default",
      "workspaces, 1, 6, myBezier",
      "border, 1, 8, default",
    },
  },
  gestures = {
    workspace_swipe = true,       -- touchpad 3-finger swipe changes workspace
  },
})
```

> **Why gesture config exists:** touchpad swipe between workspaces feels natural on a scrolling layout — set `workspace_swipe = true` and swipe horizontally to move between columns/workspaces.

### 4.5 Celestia integration (`celestia.lua`)

```lua
-- ~/.config/hypr/celestia.lua
local mainMod = "SUPER"

-- Autostart the shell
hl.on("hyprland.start", function()
  hl.exec_cmd("caelestia shell -d")
end)

-- Global shortcuts registered by Celestia (via DBus GlobalShortcuts portal)
hl.bind(mainMod .. " + Space", hl.dsp.global("caelestia:toggleLauncher"))
hl.bind(mainMod .. " + L", hl.dsp.global("caelestia:lock"))
hl.bind(mainMod .. " + P", hl.dsp.global("caelestia:picker"))
```

> **Why `hl.dsp.global`:** Celestia registers its actions through the GlobalShortcuts portal (XDPH). `hl.dsp.global("app:action")` binds Hyprland keys to those registered shortcuts. Find the exact action names with `hyprctl globalshortcuts` after launching the shell.

---

## ScrollOverview — Niri-Style Overview

[ScrollOverview](https://github.com/yayuuu/hyprland-scroll-overview) adds what Hyprland lacks out of the box: a **Niri-style overview** — zoom out to see all workspaces at once, like macOS Mission Control. It also provides a trackpad swipe gesture and a visual ALT+Tab switcher.

### 5.1 Basic configuration

```lua
-- ~/.config/hypr/hyprland.lua
hl.config({
  plugin = {
    scrolloverview = {
      gesture_distance = 300,  -- how far is the "max" for the gesture
      scale = 0.5,             -- overview scale [0.1-0.9]
      workspace_gap = 100,     -- gap between workspace cards, px
      layout = "vertical",     -- vertical or horizontal
      wallpaper = 2,           -- 0: global only, 1: per-workspace only, 2: both
      blur = true,             -- blur the main overview wallpaper only
      shadow = {
        enabled = true,
        range = 50,
      },
    },
  },
})

-- Toggle overview on all monitors with SUPER+Tab
hl.bind("SUPER + Tab", function()
  hl.plugin.scrolloverview.overview("toggle all")
end)
```

### 5.2 Keybind submap (custom overview navigation)

Defining a `scrolloverview` submap replaces the built-in keyboard navigation while the overview is open. The submap activates automatically when the overview opens:

```lua
hl.define_submap("scrolloverview", function()
  hl.bind("left",   hl.plugin.scrolloverview.navigate("left"))
  hl.bind("right",  hl.plugin.scrolloverview.navigate("right"))
  hl.bind("up",     hl.plugin.scrolloverview.navigate("up"))
  hl.bind("down",   hl.plugin.scrolloverview.navigate("down"))
  hl.bind("return", hl.plugin.scrolloverview.overview("select"))
  hl.bind("escape", hl.plugin.scrolloverview.overview("off"))
end)
```

> **Note:** while the submap is active, regular Hyprland binds are not handled by default — add `{ submap_universal = true }` to any bind that must keep working (e.g. `ALT + 1..0` workspace switching).

### 5.3 Trackpad gestures

```lua
-- 3-finger vertical swipe opens/closes the overview
hl.plugin.scrolloverview.gesture({ fingers = 3, direction = "vertical" })
-- 4-finger swipe with SUPER held, stronger scale
hl.plugin.scrolloverview.gesture({ fingers = 4, direction = "vertical", mod = "SUPER", scale = 1.5 })
```

### 5.4 Dispatchers (IPC)

All actions work from Lua or via `hyprctl dispatch` — useful for scripts, bars, launchers:

```bash
hyprctl dispatch 'hl.plugin.scrolloverview.overview("toggle")'
hyprctl dispatch 'hl.plugin.scrolloverview.navigate("left")'
hyprctl dispatch 'hl.plugin.scrolloverview.window("close")'
```

Available dispatchers: `overview` (`toggle/select/open/close/off`, with optional `all` or monitor name), `navigate` (`left/right/up/down`), `window` (`select`/`close`).

### 5.5 ALT+Tab visual switcher

ScrollOverview can act as a visual ALT+Tab — opens a compact horizontal overview and advances to the next column. The full recipe lives in the plugin's [`ALT-Tab-overview.md`](https://github.com/yayuuu/hyprland-scroll-overview/blob/new-release/docs/wiki/ALT-Tab-overview.md) — it's a Lua module (~60 lines) that tracks selection state and wraps to the first column. Summary: create `~/.config/hypr/scripts/alttab.lua` with `M.next()`, then bind:

```lua
hl.bind("ALT + Tab", function()
  hl.plugin.scrolloverview.overview("open")
  require("scripts.alttab").next()
end)
```

> **Why this plugin and not `hyprscroller`:** the scrolling **layout** is already native in 0.55+ — ScrollOverview only fills the *overview* gap (zoom-out, swipe, ALT+Tab). It's the missing piece that makes Hyprland feel like Niri.

---

## Celestia Shell Setup

Celestia is a **Quickshell**-based desktop shell. After installation:

1. **Start it once from the config** — the `hl.on("hyprland.start")` hook above autostarts it on session start.
2. **Manual start / debug:** `caelestia shell -d` (foreground) or `qs -c caelestia`.
3. **IPC via CLI** — all shell functions are scriptable:

```bash
caelestia shell lock lock          # lock screen
caelestia shell wallpaper set /path/to/img   # set wallpaper
caelestia shell mpris playPause    # media control
caelestia shell picker open        # color picker
caelestia shell -s                 # list all IPC targets/functions
```

4. **Keybinds** — if not using the full caelestia dots, wire global shortcuts yourself (see §4.5). The shell exposes drawers/notifs/lock/mpris/picker/wallpaper targets via IPC.

> **Customization:** do **not** edit files installed by the AUR package — follow the [manual installation section](https://github.com/caelestia-dots/shell#manual-installation) and clone the repo into `~/.config/quickshell/caelestia` for local tweaks.

---

## SDDM Login Manager

SDDM is the login manager — it lists Hyprland as a session automatically once `hyprland.desktop` exists (installed by the `hyprland` package into `/usr/share/wayland-sessions/`).

### Choosing a theme

SDDM themes are plentiful — any theme from the [KDE Store](https://store.kde.org/browse/cat/106/order/latest/) or AUR (`sddm-theme-*`). To set one:

```bash
# Install a theme, e.g.
yay -S sddm-theme-sugar-dark

# Point SDDM at it
sudo sh -c 'echo "[Theme]
Current=sugar-dark" > /etc/sddm.conf.d/theme.conf'
```

> **Why drop-in dir:** `/etc/sddm.conf.d/` overrides the packaged defaults without touching `/etc/sddm.conf` — survives package updates.

### Verifying the session entry

```bash
ls /usr/share/wayland-sessions/
# expect: hyprland.desktop
```

If missing (e.g. custom build), install manually:
```bash
sudo cp /usr/share/wayland-sessions/hyprland.desktop /usr/share/wayland-sessions/  # or from the repo: example/hyprland.desktop
```

---

## XWayland

**Keep XWayland enabled (default).** Hyprland pulls `xorg-xwayland` as a dependency and enables it by default — legacy X11 apps (older Electron, some GTK, games) need it.

### HiDPI XWayland (only if you have a HiDPI screen)

XWayland looks pixelated on HiDPI because Xorg can't scale. Mitigation (if you see blurry X11 apps):

```lua
-- ~/.config/hypr/hyprland.lua
hl.config({
  xwayland = {
    force_zero_scaling = true,   -- force XWayland windows unscaled (kills pixelation)
  },
})
```

> **Trade-off:** `force_zero_scaling` removes the pixelation but does **not** scale apps properly — each toolkit needs its own scaling (e.g. `hl.env("GDK_SCALE", "2")` for GTK). Only set this if pixelation actually bothers you.

### Disabling XWayland entirely (NOT recommended)

```lua
hl.config({ xwayland = { enabled = false } })
```
Only do this if you hit a specific problem (memory footprint, security) and can live without X11 apps.

---

## PCoIP Keyboard Passthrough

> **Status: work in progress.** The Niri setup uses a dual-config swap (`config-normal.kdl` ↔ `config-pcoip.kdl`) with focus-based auto-swap. Hyprland's equivalent is still being validated — the pattern will differ because Hyprland config hot-reloads and uses Lua.

Planned approach (mirroring the Niri workaround):

1. Keep two Lua config variants (`hyprland.lua` ↔ `hyprland-pcoip.lua`).
2. Watch for the PCoIP client window via `hyprctl -j events` (`windowtitle` / `activewindow` events).
3. On PCoIP focus: reload config with all `Mod+` binds removed (`hyprctl reload -c hyprland-pcoip.lua` — **verify the exact `--config` reload flag**).
4. On PCoIP close: reload the normal config.

> ⚠️ The HP PCoIP client **never requests** `zwp_keyboard_shortcuts_inhibit_v1`, so Hyprland's keyboard-shortcut inhibitor (if any) won't trigger — the config-swap approach is still required, same as Niri. **This section is a plan, not a validated recipe — test before relying on it.**

---

## Troubleshooting

### `hyprctl version` shows git build / `.so` errors

You installed `hyprland-git` or built manually. The wiki's answer: *entirely your fault* — install the official `hyprland` package instead. ABI-breaking dependency updates require recompiling everything.

### SDDM takes ~90s to shut down

Bug [#1476](https://github.com/sddm/sddm/issues/1476) — fixed in SDDM ≥ 0.20.0. Update `sddm` (`sudo pacman -Syu`) or install `sddm-git`.

### Config error on reload

- **Syntax errors** → Hyprland refuses to reload and pops an error. Fix the file, save again (hot-reload).
- **Runtime type errors** → execution continues, error is shown.
- **One file kills everything** → a `require("nonexistent")` throws in the calling file. Wrap in `pcall()` if you need resilience:
  ```lua
  local ok, err = pcall(require, "maybe-nonexistent")
  ```

### Celestia doesn't start / no bar

- Run `caelestia shell -d` in a terminal — read the error.
- Check the autostart hook fired: `hyprctl -j events` and look for startup errors.
- Quickshell must be the **git** version — `quickshell-git` (tagged release will fail to load Celestia).
- Missing fonts: Celestia needs `material-symbols` + a Nerd Font (e.g. `caskaydia-cove-nerd`).

### ScrollOverview won't load / `.so` mismatch after Hyprland update

`hyprpm` plugins are built against your exact Hyprland version. After a Hyprland update, rebuild the plugin:

```bash
hyprpm update      # rebuild + fetch dependencies
hyprpm enable scrolloverview
```

If you run a git build of Hyprland, the plugin must come from the `new-release` branch (see §3) — and expect to recompile on every compositor update.

### XWayland apps blurry on HiDPI

See §XWayland — set `force_zero_scaling = true` and per-toolkit scaling.

### Emergency keybinds after a config error

If a major error kills your binds, Hyprland provides emergency binds: <kbd>SUPER</kbd>+<kbd>Q</kbd> (terminal), <kbd>SUPER</kbd>+<kbd>R</kbd> (run), <kbd>SUPER</kbd>+<kbd>M</kbd> (exit).

---

## Uninstalling

```bash
# Stop SDDM first
sudo systemctl disable sddm --now

# Remove Hyprland + ecosystem
sudo pacman -Rns hyprland xdg-desktop-portal-hyprland hyprpolkitagent sddm

# Remove Celestia shell (AUR)
yay -Rns caelestia-shell

# Remove configs
rm -rf ~/.config/hypr ~/.config/quickshell
```

> **Why `-Rns`:** removes the packages, their now-unneeded dependencies (`-s`), and their config files (`-n`).

---

*Sources: [Hyprland wiki](https://wiki.hypr.land/) (crawled 2026-08-11 — Lua config, scrolling layout, SDDM compat, XWayland), [Celestia shell](https://github.com/caelestia-dots/shell), [SDDM bug #1476](https://github.com/sddm/sddm/issues/1476).*
