# Hyprland + Noctalia — Native Tiling 🪟🪽

Installing Hyprland (native tiling Wayland compositor — **dwindle** default, **master** optional) with the Noctalia shell on an existing Arch Linux install. Noctalia is a native C++ desktop shell — bar, launcher, notifications, lock screen, idle, wallpaper, and color picker all in one.

> **This guide targets Hyprland 0.55+.** Since 0.55, Hyprland's config language is **Lua** (`hyprland.lua`) — the old hyprlang syntax (`hyprland.conf`) is deprecated. Check your version with `hyprctl version` before following any config examples.
>
> **Prefer scrolling?** There's a companion guide — `hyprland-Scrolling.md` — for the Niri-style scrolling layout with ScrollOverview.
>
> **Sources:** [Hyprland wiki](https://wiki.hypr.land/) (crawled 2026-08-11), [Noctalia v5 docs](https://github.com/noctalia-dev/noctalia), [SDDM](https://github.com/sddm/sddm).

---

## 📖 Table of Contents

1. [Overview](#overview)
2. [Package Stack — Lean But Feature-Rich](#package-stack--lean-but-feature-rich)
3. [Installation](#installation)
4. [Configuration (Lua)](#configuration-lua)
5. [HyprMod — GUI Settings](#hyprmod--gui-settings)
6. [Noctalia Shell Setup](#noctalia-shell-setup)
7. [KDE Apps on Hyprland](#kde-apps-on-hyprland)
8. [SDDM Login Manager](#sddm-login-manager)
9. [XWayland](#xwayland)
10. [Cursor Themes](#cursor-themes)
11. [PCoIP Keyboard Passthrough](#pcoip-keyboard-passthrough)
12. [Troubleshooting](#troubleshooting)
13. [Uninstalling](#uninstalling)

---

## Overview

| Component | Package | Source | Purpose |
|-----------|---------|--------|---------|
| **Hyprland** | `hyprland` | `extra` (official) | Wayland compositor with native tiling (dwindle + master) |
| **Noctalia** | `noctalia-git` | AUR | Desktop shell: bar, launcher, notifications, lock, idle, wallpaper, OSD, clipboard |
| **SDDM** | `sddm` | `extra` (official) | Login manager (≥ 0.20.0 to avoid bug #1476) |
| **XDG Desktop Portal** | `xdg-desktop-portal-hyprland` | `extra` | Screen sharing / portal integration |
| **Polkit agent** | `hyprpolkitagent` | `extra` | GUI authentication dialogs |
| **XWayland** | `xorg-xwayland` | pulled by Hyprland | Legacy X11 app support (needed — keep enabled) |

**Why this stack:**
- **Noctalia replaces 6 separate tools** — no `hyprlock` / `hypridle` / `hyprpaper` / `hyprpicker` / `waybar` / `mako` needed. One native C++ shell covers bar, launcher, notifications, lock screen, idle, wallpaper, OSD, and clipboard. No Quickshell/QML dependency (unlike Celestia).
- **Native tiling is built in** — `dwindle` (the default) for automatic tiling, `master` as an alternative layout for a master-window + stack arrangement. No plugins needed.
- **No `hyprsunset`** — Noctalia's built-in night light covers it. No `cliphist` — built-in clipboard manager.

**Key differences from Niri + Noctalia:**
- Config is **Lua** (`~/.config/hypr/hyprland.lua`), not KDL/TOML
- Tiling is **built into the compositor**, not a shell feature
- Noctalia is the same **native C++ v5 shell** as the Niri guide — only the compositor's config syntax differs (Hyprland = Lua, Niri = KDL)
- Keyboard-driven tiling like Niri, but with the classic `hjkl`/arrow window management (no tape/columns paradigm)

---

## Package Stack — Lean But Feature-Rich

```
hyprland                      # compositor (official tagged release — NOT -git)
noctalia-git                  # shell (AUR — bar/launcher/notif/lock/idle/paper/OSD/clipboard)
kitty                         # terminal (GPU-accelerated — SUPER+T / SUPER+B binds)
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
pixie-sddm-git                # SDDM theme (AUR — Material Design 3 / Pixel UI)
```

**Deliberately NOT installed:** `hyprlock`, `hypridle`, `hyprpaper`, `hyprpicker`, `hyprsunset`, `waybar`, `mako`, `rofi`, `fuzzel` — all provided by Noctalia. Also no `cliphist` / `wlsunset` / `swayidle` / `swaylock` — Noctalia v5 has those built in.

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
sudo pacman -S xdg-desktop-portal-hyprland hyprpolkitagent sddm kitty
```

> **Why:** `xdg-desktop-portal-hyprland` (XDPH) enables screen sharing under Wayland (also required for DBus global shortcuts). `hyprpolkitagent` shows GUI auth dialogs (e.g. package managers). `sddm` is the login manager.

### 3. Install screenshot stack (native)

```bash
sudo pacman -S grim slurp satty wl-clipboard
```

> **Why:** `grim` captures the screen (Wayland-native, freedesktop), `slurp` selects a region, `satty` annotates (native GTK/Adwaita, OpenGL-accelerated, wlroots — Hyprland wiki's recommended annotation tool), `wl-clipboard` copies to the Wayland clipboard. No portal needed — this is the lean, native stack. (Flameshot works but is X11-first and relies on portal support on Wayland.)

### 4. Install Noctalia shell (AUR)

```bash
yay -S noctalia-git
```

> **AUR policy — review the PKGBUILD first** (following the [June 2026 AUR malware incident](https://archlinux.org/news/active-aur-malicious-packages-incident/) checklist): `yay -S noctalia-git --editmenu` or inspect via `yay -G noctalia-git` before building. Noctalia v5 is a **native C++** rewrite of the shell (not Quickshell/QML). It provides everything in one package: bar, launcher, control center, notifications, lock screen, idle, wallpaper, OSD, clipboard, and night light — no separate `hyprlock`/`hypridle`/`hyprpaper`/`waybar`/`mako`/`cliphist`/`wlsunset`/`swaylock` needed. Binary is `noctalia` (in `$PATH`).

### 5. Install HyprMod (GUI settings app)

```bash
yay -S hyprmod
```

> **Why:** HyprMod is a native GTK4/libadwaita settings app for Hyprland — tweak any option with **live preview**, undo with Ctrl+Z, and save/share complete configs as **profiles**. It writes only to its own `hyprland-gui.conf`, never touching your main config. Requires Python 3.12+, GTK4, libadwaita (pulled automatically on Arch).

### 6. Install Dolphin (file manager)

```bash
sudo pacman -S dolphin xdg-utils gvfs-mtp plasma-integration kded
```

> **Why Dolphin:** familiar from the KDE/Niri setup — split view, tabs, terminal panel, search. `xdg-utils` makes "Open With" work (MIME + default apps), `gvfs-mtp` adds Android/phone transfer, `plasma-integration` + `kded` give KDE apps proper theming, file dialogs, trash support, and KIO network transparency. See [KDE Apps on Hyprland](#kde-apps-on-hyprland) below for the full integration setup.

### 7. Enable SDDM

```bash
sudo systemctl enable sddm --now
```

> **Why ≥ 0.20.0:** SDDM bug [#1476](https://github.com/sddm/sddm/issues/1476) causes 90-second shutdowns. Arch's `sddm` package is current (0.21+), so this is already satisfied — just don't downgrade.

### 8. Verify

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
├── keybinds.lua          # hl.bind() — tiling binds included
├── rules.lua             # hl.window_rule + hl.workspace_rule
├── animations.lua        # hl.animation + hl.gesture
└── noctalia.lua          # shell autostart + IPC binds (lock, launcher, OSD)
```

### 4.1 Main config (`hyprland.lua`)

```lua
-- ~/.config/hypr/hyprland.lua
local mainMod = "SUPER"   -- the main mod key, used everywhere

require("keybinds")
require("rules")
require("animations")
require("noctalia")

hl.config({
  general = {
    border_size = 2,
    gaps_in = 5,
    gaps_out = 10,
    layout = "dwindle",            -- native tiling: "dwindle" (default) | "master"
    ["col.active_border"] = "0xff89b4fa",
    ["col.inactive_border"] = "0xff45475a",
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

-- Dwindle layout (default) — sensible tweaks
hl.config({
  dwindle = {
    preserve_split = true,         -- keep split direction when moving windows
  },
})
```

> **Dwindle explained:** Hyprland's default automatic tiler — new windows split the focused window (alternating split direction), the tree stays balanced, and `SUPER+Shift+Space` swaps the focused window with the master. It "just works" with zero config; the two tweaks above keep it predictable.

#### Master layout (optional alternative)

Prefer a **master window + stack** arrangement instead of dwindle's binary tree? Switch the global layout, or per-workspace via `hl.workspace_rule` (see §4.3):

```lua
-- Option A: master everywhere
hl.config({ general = { layout = "master" } })

-- Option B: master only on workspace 2 (dwindle elsewhere)
hl.workspace_rule({ workspace = "2", layout = "master", layout_opts = {
  mfact = 0.6,                     -- master takes 60% of the screen
  new_status = "slave",            -- new windows join the stack (not master)
  orientation = "left",            -- master on the left, stack on the right
} })
```

> **Master options:** `mfact` (master fraction 0–1), `new_status` (`master`/`slave`), `always_show_focus`, `orientation` (`left`/`right`/`top`/`bottom`). With master, `SUPER+Shift+Space` promotes the focused stack window to master.

### 4.2 Keybinds (`keybinds.lua`)

> **Native tiling:** classic `hjkl`/arrow window management — `SUPER+H/J/K/L` + arrows focus, `SUPER+Shift+…` move, `SUPER+Ctrl+…` resize, `SUPER+Space` launcher, `SUPER+F` fullscreen, `SUPER+Tab` cycle windows. Workspace navigation keeps the Niri-era `SUPER+U/I` + `SUPER+Shift+scroll` habits.

```lua
-- ~/.config/hypr/keybinds.lua
local mainMod = "SUPER"

--  Launch
hl.bind(mainMod .. " + T", hl.dsp.exec_cmd("kitty"))             -- terminal
hl.bind(mainMod .. " + Space", hl.dsp.exec_cmd("noctalia msg panel-toggle launcher"))  -- launcher (Noctalia)
hl.bind(mainMod .. " + B", hl.dsp.exec_cmd("kitty -e nvim"))     -- editor

--  Window management
hl.bind(mainMod .. " + Q", hl.dsp.window.close())                    -- close window
hl.bind(mainMod .. " + V", hl.dsp.window.float({ action = "toggle" }))  -- toggle floating

--  Focus window (hjkl + arrows)
hl.bind(mainMod .. " + H", hl.dsp.focus({ direction = "l" }))
hl.bind(mainMod .. " + L", hl.dsp.focus({ direction = "r" }))
hl.bind(mainMod .. " + J", hl.dsp.focus({ direction = "d" }))
hl.bind(mainMod .. " + K", hl.dsp.focus({ direction = "u" }))
hl.bind(mainMod .. " + Left", hl.dsp.focus({ direction = "l" }))
hl.bind(mainMod .. " + Right", hl.dsp.focus({ direction = "r" }))
hl.bind(mainMod .. " + Down", hl.dsp.focus({ direction = "d" }))
hl.bind(mainMod .. " + Up", hl.dsp.focus({ direction = "u" }))

--  Move window (hjkl + arrows)
hl.bind(mainMod .. " + SHIFT + H", hl.dsp.window.move({ direction = "l" }))
hl.bind(mainMod .. " + SHIFT + L", hl.dsp.window.move({ direction = "r" }))
hl.bind(mainMod .. " + SHIFT + J", hl.dsp.window.move({ direction = "d" }))
hl.bind(mainMod .. " + SHIFT + K", hl.dsp.window.move({ direction = "u" }))
hl.bind(mainMod .. " + SHIFT + Left", hl.dsp.window.move({ direction = "l" }))
hl.bind(mainMod .. " + SHIFT + Right", hl.dsp.window.move({ direction = "r" }))
hl.bind(mainMod .. " + SHIFT + Down", hl.dsp.window.move({ direction = "d" }))
hl.bind(mainMod .. " + SHIFT + Up", hl.dsp.window.move({ direction = "u" }))

--  Resize window (40px steps)
hl.bind(mainMod .. " + CTRL + H", hl.dsp.window.resize({ x = -40, y = 0 }))
hl.bind(mainMod .. " + CTRL + L", hl.dsp.window.resize({ x = 40, y = 0 }))
hl.bind(mainMod .. " + CTRL + J", hl.dsp.window.resize({ x = 0, y = 40 }))
hl.bind(mainMod .. " + CTRL + K", hl.dsp.window.resize({ x = 0, y = -40 }))

--  Swap with master (dwindle: swap window to master slot; master: promote to master)
hl.bind(mainMod .. " + SHIFT + Space", hl.dsp.window.swap({ next = true }))

--  Layout & view
hl.bind(mainMod .. " + F", hl.dsp.window.fullscreen({ action = "toggle" }))  -- fullscreen
hl.bind(mainMod .. " + SHIFT + F", hl.dsp.window.fullscreen_state({ internal = 2, client = 0, action = "toggle" }))  -- borderless fullscreen
hl.bind(mainMod .. " + C", hl.dsp.window.center())                     -- center floating window
hl.bind(mainMod .. " + P", hl.dsp.window.pin({ action = "toggle" }))   -- pin (sticky across workspaces)

--  Cycle windows (ALT+Tab is built-in; SUPER+Tab cycles tiled only)
hl.bind(mainMod .. " + Tab", hl.dsp.window.cycle_next({ next = true }))

--  Workspaces
for i = 1, 9 do
  hl.bind(mainMod .. " + " .. i, hl.dsp.focus({ workspace = i }))
  hl.bind(mainMod .. " + SHIFT + " .. i, hl.dsp.window.move({ workspace = i }))
end
hl.bind(mainMod .. " + E", hl.dsp.exec_cmd("dolphin"))             -- file manager
hl.bind(mainMod .. " + N", hl.dsp.focus({ workspace = "e+1" }))                -- next empty workspace
hl.bind(mainMod .. " + U", hl.dsp.focus({ workspace = "+1" }))                 -- next workspace
hl.bind(mainMod .. " + I", hl.dsp.focus({ workspace = "-1" }))                 -- prev workspace
hl.bind(mainMod .. " + SHIFT + U", hl.dsp.window.move({ workspace = "+1" }))   -- move window to next workspace
hl.bind(mainMod .. " + SHIFT + I", hl.dsp.window.move({ workspace = "-1" }))   -- move window to prev workspace

--  Mouse wheel (workspace navigation)
hl.bind(mainMod .. " + mouse_down", hl.dsp.focus({ workspace = "+1" }))        -- scroll down → next workspace
hl.bind(mainMod .. " + mouse_up", hl.dsp.focus({ workspace = "-1" }))          -- scroll up → prev workspace
hl.bind(mainMod .. " + SHIFT + mouse_down", hl.dsp.focus({ workspace = "e+1" }))  -- shift+scroll down → next empty
hl.bind(mainMod .. " + SHIFT + mouse_up", hl.dsp.focus({ workspace = "e-1" }))    -- shift+scroll up → prev empty

--  Screenshots
hl.bind("Print", hl.dsp.exec_cmd('grim - | satty -f -'))                      -- fullscreen → annotate
hl.bind(mainMod .. " + SHIFT + S", hl.dsp.exec_cmd('grim -g "$(slurp)" - | satty -f -'))  -- region → annotate
hl.bind("CTRL + Print", hl.dsp.exec_cmd('grim -g "$(slurp -d)" - | wl-copy'))  -- screen → clipboard

--  System
hl.bind(mainMod .. " + ALT + L", hl.dsp.exec_cmd("noctalia msg session lock"))  -- lock (Noctalia)
hl.bind(mainMod .. " + SHIFT + E", hl.dsp.exec_cmd("hyprctl dispatch exit"))    -- logout / relaunch session
hl.bind(mainMod .. " + X", hl.dsp.exec_cmd("shutdown now"))                     -- power off
```

> **Keybind callbacks must not block** — the compositor event loop runs them. Use `hl.dsp.exec_cmd(...)` for anything external; don't call `wl-paste`/`io.popen`/network I/O inside a bind function (a hung bind freezes the whole desktop).

#### Keybind summary

`SUPER` is the main mod. Tiling follows classic Hyprland (hjkl) rather than Niri's tape paradigm:

| Action | Keybind |
|--------|---------|
| Terminal | `SUPER+T` |
| Launcher (Noctalia) | `SUPER+Space` |
| Editor | `SUPER+B` |
| Close window | `SUPER+Q` |
| Toggle floating | `SUPER+V` |
| Focus window L/R/U/D | `SUPER+H/L/J/K` + arrows |
| Move window L/R/U/D | `SUPER+Shift+H/L/J/K` + arrows |
| Resize window | `SUPER+Ctrl+H/L/J/K` (40px) |
| Swap with master | `SUPER+Shift+Space` |
| Fullscreen | `SUPER+F` |
| Fake fullscreen | `SUPER+Shift+F` |
| Center floating | `SUPER+C` |
| Pin window | `SUPER+P` |
| Cycle windows | `SUPER+Tab` |
| Workspace 1–9 | `SUPER+1..9` |
| Move window to ws | `SUPER+Shift+1..9` |
| Focus ws up/down | `SUPER+U/I` (or `SUPER+scroll`) |
| Move to ws up/down | `SUPER+Shift+U/I` |
| Next empty ws | `SUPER+N` |
| File manager | `SUPER+E` |
| Screenshot full/region/clip | `Print` / `SUPER+Shift+S` / `Ctrl+Print` |
| Lock | `SUPER+Alt+L` |
| Logout / Power | `SUPER+Shift+E` / `SUPER+X` |

### 4.3 Rules (`rules.lua`)

```lua
-- ~/.config/hypr/rules.lua
-- Float certain dialogs
hl.window_rule({ match = { class = "pavucontrol" }, float = true })
hl.window_rule({ match = { title = "^(Open File)" }, float = true })

-- Center new floating windows
hl.window_rule({ match = { class = ".*" }, center = true })

-- Per-workspace layout: master on workspace 2, dwindle elsewhere (optional)
hl.workspace_rule({ workspace = "2", layout = "master", layout_opts = { mfact = 0.6 } })

-- Monitor setup (adjust to your hardware)
hl.monitor({ output = "DP-1", mode = "2560x1440@144", position = "0x0", scale = 1 })
hl.monitor({ output = "eDP-1", mode = "1920x1080@60", position = "2560x0", scale = 1 })
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
})

-- Touchpad 3-finger swipe between workspaces (Lua gesture API)
hl.gesture({ fingers = 3, direction = "horizontal", action = "workspace" })
```

> **Why gesture config exists:** touchpad swipe between workspaces works the same on tiling layouts — set `hl.gesture({ fingers = 3, direction = "horizontal", action = "workspace" })` and swipe horizontally (3 fingers) to move between workspaces.

### 4.5 Noctalia integration (`noctalia.lua`)

First, enable the **polkit authentication agent** — a user service that shows GUI auth dialogs (e.g. package manager prompts, system settings). It starts automatically with the Hyprland graphical session:

```bash
systemctl --user enable --now hyprpolkitagent
```

> **Why:** `hyprpolkitagent` ships a systemd user unit (`/usr/lib/systemd/user/hyprpolkitagent.service`, `WantedBy=graphical-session.target`) but it is **disabled by default** — without enabling it, apps that need root privileges have no GUI auth dialog. The binary lives at `/usr/lib/hyprpolkitagent/hyprpolkitagent`, not in `$PATH`.

```lua
-- ~/.config/hypr/noctalia.lua
local mainMod = "SUPER"

-- Autostart the shell (foreground; Noctalia detaches its own services)
hl.on("hyprland.start", function()
  hl.exec_cmd("noctalia")
end)
```

> **IPC is via `noctalia msg` (socket), not DBus GlobalShortcuts.** The shell runs as the `noctalia` process; keybinds send commands to the running instance with `noctalia msg <command>`. The launcher/lock/OSD binds in keybinds.lua already use these:
>
> ```bash
> noctalia msg panel-toggle launcher      # app search / calculator / emoji
> noctalia msg panel-toggle control-center
> noctalia msg settings-toggle            # GUI settings editor
> noctalia msg session lock          # lock screen
> noctalia msg volume-up / -down / -mute  # audio OSD
> noctalia msg brightness-up / -down      # brightness OSD
> ```
>
> Run `noctalia msg` (with no command) to see the full list on your install.

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

## Noctalia Shell Setup

[Noctalia](https://github.com/noctalia-dev/noctalia) v5 is a **native C++** desktop shell (bar, launcher, control center, notifications, lock screen, idle, wallpaper, OSD, clipboard, night light) — no Quickshell/QML dependency. Config is TOML under `[shell]`, with a **GUI settings editor** as the primary way to tweak it.

### First start

The shell autostarts from the `hyprland.start` hook (see §4.5). To debug or start it manually:

```bash
noctalia              # run in the foreground
noctalia -d           # run as a daemon (background)
noctalia msg          # IPC to the running instance (shows command list if none given)
```

### IPC via `noctalia msg`

All shell functions are scriptable through the `noctalia msg` socket (verified on a live v5 install):

```bash
noctalia msg panel-toggle launcher        # open/close launcher (app search / calculator / emoji)
noctalia msg panel-toggle control-center  # control center
noctalia msg settings-toggle              # GUI settings editor
noctalia msg session lock                  # lock screen
noctalia msg volume-up / -down / -mute    # audio OSD
noctalia msg brightness-up / -down        # brightness OSD
noctalia msg panel-toggle session         # session menu (logout / power)
```

> **What IPC is:** IPC (*Inter-Process Communication*) — `noctalia msg ...` sends commands over a socket to the **already-running** shell process. It does not start a new instance. This is how keybinds drive the shell: `hl.dsp.exec_cmd("noctalia msg session lock")` tells the running shell to lock the screen.
>
> **Verify before relying:** run `noctalia msg` (no command) to print the full command list on your version — command names occasionally change between beta releases.

### Config file

Config lives at `~/.config/noctalia/config.toml`. The **GUI settings editor** (`noctalia msg settings-toggle`) is the primary way to configure; only a few manual overrides are commonly needed:

```toml
# ~/.config/noctalia/config.toml
[shell]
clipboard_enabled = true        # built-in clipboard manager (no cliphist needed)
telemetry_enabled = false

[shell.panel]
transparency_mode = "solid"     # solid | soft | glass
borders = true
shadow = true

[shell.launcher]
categories = true
sort_by_usage = true
```

> The shell reloads TOML on change; the settings editor writes to this file. Options you omit fall back to their defaults.

### Live wallpapers (optional)

Noctalia's wallpaper engine handles **static images** (and the wallpaper picker reads from `~/Pictures/Wallpapers` by default). For **animated** video wallpapers, use **mpvpaper** — a video wallpaper daemon for wlroots compositors:

```bash
# Install
yay -S mpvpaper

# Download a live wallpaper (e.g. from motionbgs.com — pages expose the .mp4 directly)
curl -L -o ~/Videos/twilight-at-mount-fuji.mp4 \
  https://motionbgs.com/media/9962/twilight-at-mount-fuji.960x540.mp4

# Play it fullscreen, looping (HDMI-A-1 = your monitor)
mpvpaper -o "loop" HDMI-A-1 ~/Videos/twilight-at-mount-fuji.mp4

# Stop
pkill mpvpaper
```

**Autostart with Hyprland:**

```lua
-- ~/.config/hypr/hyprland.lua
hl.exec_cmd("mpvpaper -o 'loop' HDMI-A-1 ~/Videos/twilight-at-mount-fuji.mp4")
```

> **⚠️ Conflict with Noctalia's wallpaper:** both Noctalia's wallpaper and mpvpaper draw on the same layer-shell background layer — running both at once stacks/overlaps them. Use either mpvpaper **or** the Noctalia static wallpaper, not both simultaneously (set the Noctalia wallpaper back when you stop mpvpaper).
>
> **Performance note:** video wallpapers use a hardware-accelerated mpv context but still cost GPU/VRAM — expect a few percent GPU load while running. The `960x540` MotionBGs preview is lighter than the full-res version.

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

> **SDDM auto-unlock:** on Arch, SDDM ships with `pam_kwallet`/`pam_gnome_keyring` hooks in `/etc/pam.d/sddm`. Verify they're present (`grep -i kwallet /etc/pam.d/sddm`) — if missing, add the `auth optional` / `session optional` lines from the Niri guide's SDDM setup. KWallet auto-unlock: wallet password = login password, blowfish encryption, wallet name = `kdewallet`.

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

> **Important:** environment.d files use `KEY=VALUE` syntax and require **absolute paths**. `$HOME`, `%h`, **and `$USER` do NOT expand here** — replace `USERNAME` below with your actual username (e.g. `/home/pop/`).

```ini
QT_QPA_PLATFORM=wayland
QT_QPA_PLATFORMTHEME=kde
QT_QPA_PLATFORMTHEME_QT6=kde
QT_STYLE_OVERRIDE=breeze
XDG_MENU_PREFIX=plasma-
XDG_DATA_DIRS=/home/USERNAME/.local/share/flatpak/exports/share:/var/lib/flatpak/exports/share:/usr/local/share:/usr/share
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

This guide uses **[pixie-sddm](https://github.com/xCaptaiN09/pixie-sddm)** — a Material Design 3 / Google Pixel-inspired theme (two-tone stacked clock, dynamic color extraction from your wallpaper, blur, circular avatar, Qt6 + Qt5 support).

```bash
# Prereqs for the Qt6 engine (SDDM on Arch is Qt6)
sudo pacman -S qt6-declarative qt6-svg

# Install the theme (AUR, maintained by the theme author)
yay -S pixie-sddm-git

# Point SDDM at it
sudo sh -c 'echo "[Theme]
Current=pixie" > /etc/sddm.conf.d/theme.conf'
```

> **Why drop-in dir:** `/etc/sddm.conf.d/` overrides the packaged defaults without touching `/etc/sddm.conf` — survives package updates.
>
> **AUR policy — review the PKGBUILD first:** `pixie-sddm-git` is maintained by the theme author (xCaptaiN09); it installs only to `/usr/share/sddm/themes/pixie`. Verify with `yay -G pixie-sddm-git` if desired.
>
> **Other themes:** any SDDM theme from the [KDE Store](https://store.kde.org/browse/cat/106/order/latest/) or AUR (`sddm-theme-*`) works — just change `Current=` accordingly.

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

## Cursor Themes

Two cursor themes — install both, pick one as active:

| Theme | Style | AUR | Formats |
|-------|-------|-----|---------|
| **Phinger** | Windows 11-ish, "most likely the most over-engineered cursor theme" | `phinger-cursors` | XCursor |
| **Bibata Classic** | Material-based, bold classic outline | `bibata-cursor-git` | **Hyprcursor + XCursor** |

```bash
yay -S phinger-cursors bibata-cursor-git
```

**Hyprland native cursor (hyprcursor):** Bibata is the hyprcursor-capable one (Hyprland ≥0.37 uses hyprcursor for its own cursor):

```bash
hyprctl setcursor Bibata-Modern-Classic 24
```

> Find the exact installed theme names with `ls /usr/share/icons/` (Bibata ships `Bibata-Modern-*` / `Bibata-Original-*` variants; Hyprcursor themes also live in `/usr/share/icons/hyprcursor/`) — Bibata ships `Bibata-Modern-*` / `Bibata-Original-*` variants (Classic/Ice/Amber).

**XWayland + GTK apps** use XCursor via env vars (Hyprland also syncs the theme to gsettings when `sync_gsettings_theme` is enabled):

```lua
-- ~/.config/hypr/hyprland.lua
hl.env("XCURSOR_THEME", "phinger-cursors")   -- or "Bibata-Modern-Classic"
hl.env("XCURSOR_SIZE", "24")
```

> **Switching themes:** keep native (`hyprctl setcursor`) and `XCURSOR_THEME` pointing at the same theme so Wayland and XWayland cursors match. XWayland apps keep the xcursor lookup path — that's why `phinger-cursors` (XCursor-only) still works there.

---

## PCoIP Keyboard Passthrough

> **Status: expected to work — not yet validated on real hardware.** Much simpler than the Niri case: Hyprland ships **integrated XWayland** (pulled by the `hyprland` package) — there is no `xwayland-satellite` involved, so the config-swap hack Niri needs should **not** be required here.

The HP PCoIP client runs through the [pcoip-client](https://github.com/poppatchara/pcoip-client) fork (Arch package), which ships a **vendored Qt 6.9 Wayland client**. Two paths:

1. **Native Wayland (primary):** the wrapper (`~/.local/bin/pcoip`) prefers native Wayland when `WAYLAND_DISPLAY` is set — the client runs as a plain Wayland window. No XWayland, no config swap.
2. **XWayland fallback:** if the client falls back to X11/XCB, Hyprland's built-in XWayland handles it as a normal X11 window — **no `xwayland-satellite` daemon** (unlike Niri), so no focus-based config swapping is needed.

**Keyboard shortcuts:** Hyprland supports the keyboard-shortcuts-inhibit protocol. The HP client still never requests `zwp_keyboard_shortcuts_inhibit_v1` (same as Niri), but if a specific `SUPER+` bind interferes while the client is focused, two levers exist:

- bind option `dont_inhibit = true` — bypasses the app's inhibit requests for that bind
- window rule `no_shortcuts_inhibit = true` — disallows the app from inhibiting your shortcuts

> **Test checklist on the real HP Anyware session:** ① client launches under native Wayland (`WAYLAND_DISPLAY` set in the wrapper); ② typing reaches the remote desktop (no local bind swallows keys); ③ if a bind interferes, add a `hl.window_rule({ match = { class = "..." }, no_shortcuts_inhibit = true })` for the client class rather than swapping the whole config.

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

### Noctalia doesn't start / no bar

- Run `noctalia` in a terminal — read the error (it prints a clear reason, e.g. missing config or a Wayland/socket issue).
- Check the autostart hook fired: `hyprctl -j events` and look for startup errors.
- Make sure `noctalia` is started **after** Hyprland (the `hyprland.start` hook in §4.5) — it needs a compositor and the Wayland socket.
- Missing fonts: Noctalia expects a Nerd Font and `material-symbols`-style icon font on your system (install e.g. `noto-fonts` + a Nerd Font package if glyphs/boxes render).

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

# Remove Noctalia shell (AUR)
yay -Rns noctalia-git

# Remove configs
rm -rf ~/.config/hypr ~/.config/noctalia
```

> **Why `-Rns`:** removes the packages, their now-unneeded dependencies (`-s`), and their config files (`-n`).

---

*Sources: [Hyprland wiki](https://wiki.hypr.land/) (crawled 2026-08-11 — Lua config, native tiling layouts, SDDM compat, XWayland), [Noctalia v5](https://github.com/noctalia-dev/noctalia), [SDDM bug #1476](https://github.com/sddm/sddm/issues/1476).*
