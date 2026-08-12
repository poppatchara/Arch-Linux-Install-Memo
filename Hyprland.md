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
6. [HyprMod — GUI Settings](#hyprmod--gui-settings)
7. [Celestia Shell Setup](#celestia-shell-setup)
8. [Walker — Application Launcher](#walker--application-launcher)
9. [KDE Apps on Hyprland](#kde-apps-on-hyprland)
10. [SDDM Login Manager](#sddm-login-manager)
11. [XWayland](#xwayland)
12. [PCoIP Keyboard Passthrough](#pcoip-keyboard-passthrough)
13. [Troubleshooting](#troubleshooting)
14. [Uninstalling](#uninstalling)

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
walker elephant               # launcher + backend daemon (AUR — Wayland-native, GTK4/Rust)
grim slurp satty              # screenshots: capture + region select + native annotation
wl-clipboard                  # Wayland clipboard (screenshot copy)
hyprmod                       # GUI settings app (GTK4/libadwaita) — live tweak + profiles
dolphin                       # file manager (KDE — familiar, feature-rich)
xdg-utils                     # MIME type & default apps (xdg-open, xdg-mime)
gvfs-mtp                      # Android/phone file transfer (mount MTP in Dolphin)
plasma-integration kded       # KDE file dialogs + Qt platform theme (for Dolphin)
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

### 6. Install HyprMod (GUI settings app)

```bash
yay -S hyprmod
```

> **Why:** HyprMod is a native GTK4/libadwaita settings app for Hyprland — tweak any option with **live preview**, undo with Ctrl+Z, and save/share complete configs as **profiles**. It writes only to its own `hyprland-gui.conf`, never touching your main config. Requires Python 3.12+, GTK4, libadwaita (pulled automatically on Arch).

### 7. Install Dolphin (file manager)

```bash
sudo pacman -S dolphin xdg-utils gvfs-mtp plasma-integration kded
```

> **Why Dolphin:** familiar from the KDE/Niri setup — split view, tabs, terminal panel, search. `xdg-utils` makes "Open With" work (MIME + default apps), `gvfs-mtp` adds Android/phone transfer, `plasma-integration` + `kded` give KDE apps proper theming, file dialogs, trash support, and KIO network transparency. See [KDE Apps on Hyprland](#kde-apps-on-hyprland) below for the full integration setup.

### 8. Install Walker (application launcher)

```bash
yay -S walker elephant
# elephant = backend daemon (data providers); walker = the launcher UI
systemctl --user enable --now elephant
```

> **Why Walker:** a fast, Wayland-native launcher (GTK4 layer shell + Rust) by the same author as `hyprscroller`-era tooling. Providers built in: desktop applications, calculator (`=`), file browser (`/`), command runner (`>`), websearch, clipboard history (`:`), symbol picker (`.`), provider list (`;`), Arch package search, window list, and more. Celestia's built-in launcher still exists — Walker just replaces the `SUPER+Space` keybind for a richer, faster launcher.
>
> **AUR policy — review the PKGBUILD first:** `walker` is maintained by the project author (benz, 26 votes) — inspect via `yay -G walker elephant` if you want to verify before building.

### 9. Enable SDDM

```bash
sudo systemctl enable sddm --now
```

> **Why ≥ 0.20.0:** SDDM bug [#1476](https://github.com/sddm/sddm/issues/1476) causes 90-second shutdowns. Arch's `sddm` package is current (0.21+), so this is already satisfied — just don't downgrade.

### 10. Verify

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

> **Niri-style:** these binds mirror the Niri muscle memory — `SUPER+T` terminal, `SUPER+Space` launcher, `SUPER+H/J/K/L` + arrows for column/window focus, `SUPER+Home/End` first/last column, `SUPER+Ctrl+H/L` move column, `SUPER+F`/`SUPER+Shift+F` maximize/fullscreen, `SUPER+C` center, `SUPER+E` file manager, `SUPER+Tab` overview, `SUPER+Shift+S` region screenshot, `Super+Alt+L` lock, `SUPER+Shift+E` logout.

```lua
-- ~/.config/hypr/keybinds.lua
local mainMod = "SUPER"

--  Launch (Niri: Mod+T terminal, Mod+Space launcher)
hl.bind(mainMod .. " + T", hl.dsp.exec_cmd("ghostty"))             -- terminal
hl.bind(mainMod .. " + Space", hl.dsp.exec_cmd("walker"))          -- launcher (Walker)
hl.bind(mainMod .. " + B", hl.dsp.exec_cmd("ghostty -e nvim"))     -- editor

--  Window management (Niri: Mod+Q close, Mod+V float)
hl.bind(mainMod .. " + Q", hl.dsp.killactive())                    -- close window
hl.bind(mainMod .. " + V", hl.dsp.window.float({ action = "toggle" }))  -- toggle floating

--  Scrolling layout: focus columns (Niri: Mod+H/L/Home/End, arrows)
hl.bind(mainMod .. " + H", hl.dsp.layout("focus l"))
hl.bind(mainMod .. " + L", hl.dsp.layout("focus r"))
hl.bind(mainMod .. " + Left", hl.dsp.layout("focus l"))
hl.bind(mainMod .. " + Right", hl.dsp.layout("focus r"))
hl.bind(mainMod .. " + Home", hl.dsp.layout("fit tobeg"))          -- first column
hl.bind(mainMod .. " + End", hl.dsp.layout("fit toend"))           -- last column

--  Scrolling layout: focus window up/down within column (Niri: Mod+J/K, arrows)
hl.bind(mainMod .. " + J", hl.dsp.focus({ direction = "d" }))
hl.bind(mainMod .. " + K", hl.dsp.focus({ direction = "u" }))
hl.bind(mainMod .. " + Down", hl.dsp.focus({ direction = "d" }))
hl.bind(mainMod .. " + Up", hl.dsp.focus({ direction = "u" }))

--  Scrolling layout: move column l/r (Niri: Mod+Ctrl+H/L = move-column)
hl.bind(mainMod .. " + Ctrl + H", hl.dsp.layout("swapcol l"))
hl.bind(mainMod .. " + Ctrl + L", hl.dsp.layout("swapcol r"))
hl.bind(mainMod .. " + Ctrl + Left", hl.dsp.layout("swapcol l"))
hl.bind(mainMod .. " + Ctrl + Right", hl.dsp.layout("swapcol r"))

--  Scrolling layout: scroll tape
hl.bind(mainMod .. " + period", hl.dsp.layout("move +col"))        -- scroll right by column
hl.bind(mainMod .. " + comma", hl.dsp.layout("move -col"))         -- scroll left by column

--  Scrolling layout: column width + fit (Niri: Mod+R preset, Mod+F maximize, Mod+Shift+F fullscreen, Mod+C center)
hl.bind(mainMod .. " + R", hl.dsp.layout("colresize +conf"))       -- cycle width presets
hl.bind(mainMod .. " + F", hl.dsp.layout("fit expand"))            -- maximize window to free space
hl.bind(mainMod .. " + Shift + F", hl.dsp.window.fullscreen({ action = "toggle", layout_aware = true }))  -- fullscreen
hl.bind(mainMod .. " + C", hl.dsp.layout("fit_into_view"))         -- center/fit active column into view

--  Workspaces
hl.bind(mainMod .. " + Tab", function()
  hl.plugin.scrolloverview.overview("toggle all")   -- Niri-style overview
end)
for i = 1, 9 do
  hl.bind(mainMod .. " + " .. i, hl.workspace(i))
  hl.bind(mainMod .. " + SHIFT + " .. i, hl.dsp.movetoworkspace(i))
end
hl.bind(mainMod .. " + E", hl.dsp.exec_cmd("dolphin"))             -- file manager (Niri: Mod+E)
hl.bind(mainMod .. " + N", hl.dsp.workspace("e+1"))                -- next empty workspace

--  Screenshots (Niri: Print full, Mod+Shift+S region)
hl.bind("Print", hl.dsp.exec_cmd('grim - | satty -f -'))                      -- fullscreen → annotate
hl.bind(mainMod .. " + Shift + S", hl.dsp.exec_cmd('grim -g "$(slurp)" - | satty -f -'))  -- region → annotate
hl.bind("CTRL + Print", hl.dsp.exec_cmd('grim -g "$(slurp -d)" - | wl-copy'))  -- screen → clipboard

--  System (Niri: Super+Alt+L lock, Mod+Shift+E logout)
hl.bind(mainMod .. " + Alt + L", hl.dsp.exec_cmd("caelestia shell lock lock"))  -- lock (Celestia)
hl.bind(mainMod .. " + Shift + E", hl.dsp.exec_cmd("hyprctl dispatch exit"))    -- logout / relaunch session
hl.bind(mainMod .. " + X", hl.dsp.exec_cmd("shutdown now"))                     -- power off
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

#### Optional: snappy-switcher (richer thumbnails)

The built-in ALT+Tab shows the workspace tape. If you want **big per-window thumbnails with context grouping**, add [snappy-switcher](https://github.com/OpalAayan/snappy-switcher) — pure C, Wayland layer shell, zero Electron/GTK deps:

```bash
yay -S snappy-switcher
```

Autostart the daemon and bind ALT+Tab:

```lua
hl.exec("snappy-switcher --daemon")
hl.bind("ALT + Tab", hl.dsp.exec_cmd("snappy-switcher next --mod alt"))
```

Features: context grouping, 15 themes, dismiss-on-release, MRU ordering, `--workspace` mode.

> **Default choice:** ScrollOverview ALT+Tab (§5.5) — no extra package, same visual language as the overview. Install snappy-switcher only if you want richer thumbnails; it **replaces** the ALT+Tab bind, so remove the ScrollOverview alttab module (or rebind it) to avoid conflicts. Also avoid snappy's `--workspace` mode on `SUPER+Tab` — that collides with the overview toggle.

---

## HyprMod — GUI Settings

[HyprMod](https://github.com/BlueManCZ/hyprmod) is a native GTK4/libadwaita settings app for Hyprland. It edits your running compositor **live** — tweak any option, see it change instantly, undo with Ctrl+Z, and save/share complete configurations as `.conf` profiles.

### What it manages

- **General / decoration / animation** options with live preview
- **Bezier curve editor** — control points + live animation preview
- **Monitor layout editor** — multi-monitor preview, VRR, 10-bit, HDR controls with safe defaults
- **Keybind editor** — interactive key capture, incl. mouse-drag (`bindm`) binds
- **Window / layer / workspace rules** editors with live preview
- **Autostart** (`exec` / `exec-once`) and environment variables
- **Lua config support** — migrate and edit `hyprland.lua` (0.55+) directly
- **Pending Changes page** — review every unsaved edit before saving
- **Config DNA** — a unique visual fingerprint per profile

> **Why it's safe:** HyprMod writes only to its own `hyprland-gui.conf`, never touching your main `hyprland.lua`. Profiles are plain `.conf` files you can save, name, and share.

### CLI — switch profiles from a keybind

```bash
hyprmod profile list            # show saved profiles (active marked with *)
hyprmod profile apply Gaming    # switch to a profile by name (case-insensitive)
hyprmod profile next            # cycle to the next profile (alphabetical, wraps)
hyprmod profile previous        # cycle to the previous one
```

Bind it to a key:

```lua
-- ~/.config/hypr/keybinds.lua
hl.bind(mainMod .. " + M", hl.dsp.exec_cmd("hyprmod"))   -- open settings GUI
```

> **Why the GUI only:** HyprMod is a settings tool — tweak options with live preview, save/share profiles. It is **not** used for runtime layout switching (layout stays fixed per workspace via `hl.workspace_rule`).

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

## Walker — Application Launcher

[Walker](https://github.com/abenz1267/walker) is a fast, Wayland-native application launcher (GTK4 layer shell + Rust, GPLv3). It replaces Celestia's built-in launcher on `SUPER+Space`. It needs the **elephant** backend daemon running (installed + enabled in §3 step 8).

### Providers (built-in)

| Prefix | Provider | Example |
|--------|----------|---------|
| *(none)* | Desktop applications | `firefox` |
| `=` | Calculator / unit conversion | `= 42*3` |
| `/` | File browser | `/home/pop/` |
| `>` | Command runner | `> systemctl --user restart` |
| `;` | Provider list | `;` |
| `:` | Clipboard history | `:` |
| `.` | Symbol / emoji picker | `.` |
| — | Websearch, Arch package search (`i:`), windows, bookmarks, bluetooth, snippets, todo, dmenu | — |

### Configuration

Config lives at `~/.config/walker/config.toml` (TOML, not JSON). A minimal sane starting point:

```toml
# ~/.config/walker/config.toml
single_click_activation = true
selection_wrap = true
exact_search_prefix = "'"   # disable fuzzy search with ' prefix

[providers]
default = ["desktopapplications", "calc", "websearch", "runner"]
empty = ["desktopapplications"]
```

Keybinds inside Walker: `Escape` close, `Down`/`Up` next/previous, `Page_Down`/`Page_Up` jump, `ctrl e` exact search, `alt j` actions, `F1`–`F4` quick activate.

### Service

elephant runs as a user service (starts with the graphical session):

```bash
systemctl --user enable --now elephant   # already done in §3 step 8
systemctl --user status elephant         # verify
```

> **Why Walker over Celestia's launcher:** Celestia's launch is a simple app grid; Walker adds calculator, file browsing, clipboard history, symbol picker, runner, and Arch package search — all in one fuzzy-search box. If you prefer the Celestia launcher, just revert the §4.2 bind to `caelestia shell launch`.

---

## KDE Apps on Hyprland

KDE apps (Dolphin, Kate, Okular) run fine standalone, but they need background services for full functionality — file dialogs, theme consistency, trash support, network transparency.

### Required Services

```bash
sudo pacman -S plasma-integration kded
```

| Package | Provides | What breaks without it |
|---------|----------|----------------------|
| `plasma-integration` | Qt Platform Theme | KDE apps use ugly fallback theme, wrong fonts & colors |
| `kded` | KDE Daemon (`kded6`) | KDE file dialogs fail, mime types wrong, no trash support, `kioclient` broken |

### Autostart KDE Daemon

Add to `~/.config/hypr/keybinds.lua` (or your autostart file):

```lua
hl.exec("kded6")   -- KDE daemon (file dialogs, trash, KIO)
```

> **Plasma 5 → 6:** Use `kded6` for Plasma 6 (Arch default). For older Plasma 5 systems: `kded5`.

### What KDE Apps Gain

| Feature | Without Services | With `plasma-integration` + `kded6` |
|---------|-----------------|--------------------------------------|
| **Theme** | Qt fallback (ugly) | Breeze theme, colors, fonts |
| **File Dialogs** | GTK fallback or broken | Native KDE file picker |
| **Trash** | Delete only (no restore) | Trash support with restore |
| **Network** | Some KIO fails | Full KIO (sftp, fish, smb) |
| **Mime types** | Generic icons | Proper file type icons |
| **KWallet** | May prompt endlessly | Integrated password storage |

### Secret Storage

KDE apps (Dolphin network passwords, KDE Connect) need KWallet. GTK apps (VS Code, Chromium, Firefox) use `libsecret` → the Secret Service DBus API. **KWallet alone covers both** — it ships a Secret Service interface, so GTK apps can use it as their provider. No GNOME Keyring needed (lean):

```bash
sudo pacman -S kwallet kwalletmanager kwallet-pam libsecret
```

> **Enable the Secret Service interface:** open KWallet settings (System Settings → KDE Wallet) and check **"Enable the KWallet Secret Service interface"** — otherwise GTK apps can't see the wallet. (GNOME Keyring is only needed for `gnome-online-accounts` or if you want a separate secret store.)

> **SDDM auto-unlock:** unlike the Niri guide's greetd setup, SDDM ships with `pam_kwallet`/`pam_gnome_keyring` hooks in `/etc/pam.d/sddm` on Arch. Verify they're present (`grep -i kwallet /etc/pam.d/sddm`) — if missing, add the same `auth optional` / `session optional` lines from the Niri guide. KWallet auto-unlock: wallet password = login password, blowfish encryption, wallet name = `kdewallet`.

### Dolphin "Open With" Blank Popup Fix

The most common issue: Dolphin shows an empty "Choose Application" dialog when opening files. This happens because KDE 6 renamed `applications.menu` to `plasma-applications.menu`, and `kbuildsycoca6` can't find it without `XDG_MENU_PREFIX=plasma-`.

**Root cause:** `kbuildsycoca6` — the KDE system configuration cache — is what populates Dolphin's "Open With" list. Without the menu file, it builds an empty database. The `plasma-applications.menu` file ships with `plasma-workspace` and lives at `/etc/xdg/menus/`.

**The fix requires three things:**

1. Set `XDG_MENU_PREFIX=plasma-` in your environment (so KDE looks for `plasma-applications.menu` instead of `applications.menu`)

2. Rebuild the KDE cache:

```bash
XDG_MENU_PREFIX=plasma- kbuildsycoca6 --noincremental
```

3. Auto-start `kded6` (already done above).

**Environment variables must go in TWO places** — `~/.config/environment.d/` (so systemd/portals see them) AND Hyprland's Lua config via `hl.env()` (so compositor-spawned apps see them). Systemd user services and portals can't read the compositor's env.

Create `~/.config/environment.d/10-kde-on-hyprland.conf`:

> **Important:** environment.d files use `KEY=VALUE` syntax and require **absolute paths**. `$HOME` and `%h` do NOT expand here.

```ini
QT_QPA_PLATFORM=wayland
QT_QPA_PLATFORMTHEME=kde
QT_QPA_PLATFORMTHEME_QT6=kde
QT_STYLE_OVERRIDE=breeze
XDG_MENU_PREFIX=plasma-
XDG_DATA_DIRS=/home/$USER/.local/share/flatpak/exports/share:/var/lib/flatpak/exports/share:/usr/local/share:/usr/share
QT_AUTO_SCREEN_SCALE_FACTOR=1
QT_ENABLE_HIGHDPI_SCALING=1
QT_SCALE_FACTOR_ROUNDING_POLICY=RoundPreferFloor
GTK_USE_PORTAL=1
```

Mirror the same in Hyprland's Lua config (`~/.config/hypr/environment.lua` or top of `hyprland.lua`):

```lua
hl.env("QT_QPA_PLATFORM", "wayland")
hl.env("QT_QPA_PLATFORMTHEME", "kde")
hl.env("QT_QPA_PLATFORMTHEME_QT6", "kde")
hl.env("QT_STYLE_OVERRIDE", "breeze")
hl.env("XDG_MENU_PREFIX", "plasma-")
hl.env("QT_AUTO_SCREEN_SCALE_FACTOR", "1")
hl.env("QT_ENABLE_HIGHDPI_SCALING", "1")
hl.env("QT_SCALE_FACTOR_ROUNDING_POLICY", "RoundPreferFloor")
hl.env("GTK_USE_PORTAL", "1")
```

| Variable | Why |
|----------|-----|
| `QT_QPA_PLATFORMTHEME=kde` | Use KDE's `plasma-integration` for theming (NOT `gtk3`/`qt6ct`) — lets `systemsettings` control Qt themes |
| `QT_STYLE_OVERRIDE=breeze` | Forces Breeze widget style even when Plasma isn't running |
| `XDG_MENU_PREFIX=plasma-` | **Critical:** tells `kbuildsycoca6` to use `/etc/xdg/menus/plasma-applications.menu` — without this, Dolphin's "Open With" is empty |
| `GTK_USE_PORTAL=1` | Forces GTK apps to use the portal file picker (so they respect your KDE picker preference) |
| `XDG_DATA_DIRS` | Exposes Flatpak apps to KDE the same way Plasma does |

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
