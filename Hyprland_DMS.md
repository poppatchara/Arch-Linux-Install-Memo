# Hyprland + DankMaterialShell (DMS) 🪟🎨

Installing Hyprland (0.55+, **Lua** config) with **DankMaterialShell (DMS) 1.5 "The Wolverine"** — a Material 3 desktop shell — on an existing Arch Linux install. DMS replaces waybar, swaylock, swayidle, mako, fuzzel, polkit, and everything else you'd normally stitch together into a desktop.

> **Pick one shell.** The other Hyprland companions use **Noctalia/Caelestia** (`Hyprland-Tiling.md`, `Hyprland-Scrolling.md`) — DMS is the official `dms-shell-hyprland` alternative with a Material 3 design and matugen-driven theming. Don't install two shells.
>
> **This guide targets Hyprland 0.55+ / DMS 1.5.** Since 0.55 the config language is **Lua** (`~/.config/hypr/hyprland.lua`) — the old hyprlang `hyprland.conf` still works for a few releases but is deprecated. DMS 1.5 generates native **Lua** config fragments (older docs showing `.conf` snippets are pre-1.5). Check versions: `hyprctl version` and `dms --version`.
>
> **Sources:** [DMS docs — Installation](https://danklinux.com/docs/dankmaterialshell/installation), [DMS docs — Compositor Setup](https://danklinux.com/docs/dankmaterialshell/compositor-setup), [DMS docs — Keybinds & IPC](https://danklinux.com/docs/dankmaterialshell/keybinds-ipc), [DMS docs — Application Theming](https://danklinux.com/docs/dankmaterialshell/application-themes), [DMS 1.5 release notes](https://danklinux.com/blog/v1-5-release), [Hyprland wiki](https://wiki.hypr.land/), [Hyprland 0.55 announcement](https://hypr.land/news/update55).

---

## 📖 Table of Contents

1. [Overview](#overview)
2. [Package Stack — Official Packages Only](#package-stack--official-packages-only)
3. [Installation](#installation)
4. [Hyprland Configuration (Lua)](#hyprland-configuration-lua)
5. [DMS Shell Setup](#dms-shell-setup)
6. [Keybinds & IPC](#keybinds--ipc)
7. [Application Theming (matugen)](#application-theming-matugen)
8. [Qt/KDE App Theming](#qtkde-app-theming)
9. [Login Manager (SDDM / DankGreeter)](#login-manager-sddm--dankgreeter)
10. [Troubleshooting](#troubleshooting)
11. [Uninstalling](#uninstalling)

---

## Overview

| Component | Package | Source | Purpose |
|-----------|---------|--------|---------|
| **Hyprland** | `hyprland` | `extra` (official) | Wayland compositor, native tiling, Lua config (0.55+) |
| **DMS** | `dms-shell-hyprland` | `extra` (official) | Desktop shell: bar, launcher, notifications, control center, lock+idle, clipboard, wallpaper, night mode |
| **Quickshell** | `quickshell` | pulled by `dms-shell` | Core framework DMS is built on (only hard dependency) |
| **matugen** | `matugen` | `extra` (official) | Material You color generation — DMS themes everything from the wallpaper |
| **XDG Desktop Portal** | `xdg-desktop-portal-hyprland` | `extra` (official) | Screen sharing / portal integration |
| **Polkit agent** | `hyprpolkitagent` | `extra` (official) | GUI authentication dialogs |
| **SDDM** | `sddm` | `extra` (official) | Login manager (or DMS's own **DankGreeter**) |
| **XWayland** | `xorg-xwayland` | pulled by Hyprland | Legacy X11 app support (keep enabled) |

**Why this stack:**
- **DMS replaces 7 separate tools** — `waybar` / `swaylock` / `swayidle` / `mako` / `fuzzel` / `polkit` / `hyprpaper`. It even has a built-in night mode (gamma/color-temperature with suncalc) and wallpaper manager — no `hyprsunset`, no `wlsunset`.
- **Everything official** — `hyprland`, `dms-shell-hyprland`, `matugen`, `xdg-desktop-portal-hyprland`, `hyprpolkitagent` all ship in `extra`. No AUR, no `-git`.
- **Material 3 theming is automatic** — matugen derives a color scheme from the wallpaper on every wallpaper change; DMS propagates it to the shell, GTK, Qt, terminals (dank16 palette), and your own templates.
- **Hyprland 0.55+ Lua config** — modular `hl.config({})` blocks, `require()` per-file structure.

**Key differences from the Noctalia/Caelestia companions:**
- Shell is **Quickshell/QML** (Material 3), not native C++ / Caelestia
- Config language is **Lua** (`hyprland.lua`) — same as all 0.55+ Hyprland guides
- Theming is **matugen-driven from wallpaper**, with `dank16` ANSI palettes for terminals

---

## Package Stack — Official Packages Only

```
hyprland                      # compositor (official tagged release — NOT -git)
dms-shell-hyprland            # DMS + Hyprland integration (pulls dms-shell + hyprland)
quickshell                    # pulled by dms-shell (required framework)
matugen                       # color scheme engine (DMS execs it at runtime)
kitty                         # terminal (GPU-accelerated — SUPER+T bind)
dolphin                       # file manager (KDE; optional)
firefox                       # browser (optional)
xdg-desktop-portal-hyprland   # screen share / portals
hyprpolkitagent               # polkit GUI auth
sddm                          # login manager (or DankGreeter — see §9)
ttf-material-symbols-variable # DMS icon font
ttf-jetbrains-mono-nerd       # Nerd Font (DMS UI font)
```

**DMS optional dependencies** (from official docs — enable specific features, all optional except `quickshell`):

| Package | Enables |
|---------|---------|
| `cava` | Audio visualizer widget |
| `dankcalendar` | Desktop calendar with native DMS calendar widgets |
| `dankcalendar` (dcal) | Calendar integration |
| `dgop` | System telemetry for resource widgets |
| `dsearch` | Filesystem search engine |
| `qt6-multimedia` | System sound feedback |

---

## Installation

```bash
# 1. Compositor + DMS (official extra)
sudo pacman -S --noconfirm --needed hyprland dms-shell-hyprland

# 2. Optional DMS feature deps
sudo pacman -S --noconfirm --needed matugen cava dgop dsearch qt6-multimedia

# 3. Ecosystem
sudo pacman -S --noconfirm --needed kitty dolphin xdg-desktop-portal-hyprland \
  hyprpolkitagent ttf-material-symbols-variable ttf-jetbrains-mono-nerd

# 4. Verify
hyprctl version          # expect a tagged release (0.55+)
dms --version            # expect 1.5.x
```

### Systemd User Session (required — official DMS setup)

Hyprland does **not** initialize the systemd user session by default, so DMS (a systemd user service) won't start. From the [official DMS installation docs](https://danklinux.com/docs/dankmaterialshell/installation):

**1. Create a session target** — `~/.config/systemd/user/hyprland-session.target`:

```ini
[Unit]
Description=Hyprland Session Target
Requires=graphical-session.target
After=graphical-session.target
```

**2. Export the environment + start the target from Hyprland** — add to `~/.config/hypr/hyprland.lua` (after any `env` lines):

```lua
hl.on("hyprland.start", function()
  hl.exec_cmd("dbus-update-activation-environment --systemd --all")
  hl.exec_cmd("systemctl --user start hyprland-session.target")
end)
```

**3. Bind DMS to the session** — DMS starts with Hyprland, stops when it exits, and won't run in other sessions (Plasma, GNOME, etc.):

```bash
systemctl --user add-wants hyprland-session.target dms
```

> **If you use `dms-greeter`** (DankGreeter), seat management, D-Bus initialization, and runtime dirs are handled automatically — skip the manual steps above. See §9.

---

## Hyprland Configuration (Lua)

Hyprland 0.55+ reads `~/.config/hypr/hyprland.lua`. DMS 1.5 generates its **own Lua fragments** when you run `dms setup` (or via `dankinstall`) — colors, layout, outputs, key binds. Keep your `hyprland.lua` minimal and let DMS manage what it owns.

### 4.1 Main config (`hyprland.lua`)

```lua
-- ~/.config/hypr/hyprland.lua
local mainMod = "SUPER"

-- DMS-managed fragments (generated by `dms setup` — colors, layout, outputs)
require("dms.colors")
require("dms.layout")
require("dms.outputs")

hl.config({
  general = {
    gaps_in = 5,
    gaps_out = 5,
    border_size = 0,                 -- DMS draws its own borders via the shell
    layout = "dwindle",              -- native tiling
  },
  decoration = {
    rounding = 12,
    active_opacity = 1.0,
    inactive_opacity = 0.9,
    shadow = { enabled = true, range = 30, render_power = 5, offset = "0 5", color = "0x70000000" },
  },
  misc = {
    disable_hyprland_logo = true,    -- hide the default Hyprland backdrop
    disable_splash_rendering = true,
  },
})
```

> **DMS auto-applies gaps, window radius, and colors** — customize them in **Settings → Compositor**, not by hand-editing `dms/*.lua` (DMS owns those files and regenerates them).

### 4.2 Environment variables

```lua
hl.env("QT_QPA_PLATFORM", "wayland")
hl.env("ELECTRON_OZONE_PLATFORM_HINT", "auto")
hl.env("QT_QPA_PLATFORMTHEME", "gtk3")        -- DMS official default (see §8 for KDE apps)
hl.env("QT_QPA_PLATFORMTHEME_QT6", "gtk3")
```

### 4.3 Layer rules

```lua
hl.layer_rule("no_anim on", "match:namespace ^(dms)$")
```

### 4.4 Window rules

```lua
-- Inactive window opacity
hl.window_rule("opacity 0.9 0.9", "match:float 0, match:focus 0")
-- GNOME apps — rounded, borderless
hl.window_rule("rounding 12, border_size 0", "match:class ^(org\\.gnome\\.)")
-- Terminals — no borders
hl.window_rule("border_size 0", "match:class ^(Alacritty)$")
hl.window_rule("border_size 0", "match:class ^(com\\.mitchellh\\.ghostty)$")
hl.window_rule("border_size 0", "match:class ^(kitty)$")
-- DMS windows float by default
hl.window_rule("float on", "match:class ^(org.quickshell)$")
```

> **Rule syntax:** Hyprland 0.55+ Lua keeps the hyprlang rule *strings* (`hl.window_rule(rule, match)`); the old `windowrule = rule, match` lines map 1:1.

### 4.5 Autostart

With the systemd session setup (above), DMS starts via `dms.service` — you don't need `exec-once = dms run` in Lua. Add other autostarts either as systemd user units or:

```lua
hl.on("hyprland.start", function()
  hl.exec_cmd("waybar")   -- only if you still run extra bars (DMS replaces it)
end)
```

---

## DMS Shell Setup

### First start

1. Launch Hyprland. DMS starts automatically via the systemd session target.
2. On first run DMS walks you through setup: monitor layout, wallpaper, theme.
3. `dms setup` (re)generates the Lua fragments + writes configs — run it after installing new DMS-managed compositors.

### Config locations

| Path | What |
|------|------|
| `~/.config/DankMaterialShell/settings.json` | DMS settings (theme mode, templates, qtTheming…) |
| `~/.config/DankMaterialShell/plugins.lock.json` | Plugin registry (managed installs, pinned commits) |
| `~/.local/state/DankMaterialShell/session.json` | Session state (light/dark mode, wallpaper) |
| `~/.cache/DankMaterialShell/` | Generated colors + runtime data |
| `~/.config/hypr/dms/` | Hyprland Lua fragments DMS manages |

### Built-in features (no extra tools)

- **Wallpaper** — `dms ipc call wallpaper set …` or the wallpaper browser (`Mod+Y`)
- **Night mode** — gamma/color-temperature with suncalc scheduling (`dms ipc call night …`)
- **Lock + idle** — DMS bundles both (`dms ipc call lock lock`)
- **Clipboard** — built-in history manager
- **Screenshots** — `dms ipc call screenshot …`
- **Brightness / audio** — `dms ipc call brightness …` / `dms ipc call audio …`

---

## Keybinds & IPC

All DMS control flows through `dms ipc call <target> <function> [params…]` ([official reference](https://danklinux.com/docs/dankmaterialshell/keybinds-ipc)).

**Core binds** (DMS default keybinds — `$mod = SUPER`):

```lua
-- Application launchers
hl.bind(mainMod .. " + space", hl.dsp.exec_cmd("dms ipc call spotlight toggle"))
hl.bind(mainMod .. " + V",     hl.dsp.exec_cmd("dms ipc call clipboard toggle"))
hl.bind(mainMod .. " + M",     hl.dsp.exec_cmd("dms ipc call processlist focusOrToggle"))
hl.bind(mainMod .. " + comma", hl.dsp.exec_cmd("dms ipc call settings focusOrToggle"))
hl.bind(mainMod .. " + N",     hl.dsp.exec_cmd("dms ipc call notifications toggle"))
hl.bind(mainMod .. " + Y",     hl.dsp.exec_cmd("dms ipc call dankdash wallpaper"))
hl.bind(mainMod .. " + TAB",   hl.dsp.exec_cmd("dms ipc call hypr toggleOverview"))

-- Security
hl.bind(mainMod .. " + ALT + L", hl.dsp.exec_cmd("dms ipc call lock lock"))

-- Media keys
hl.bind(", XF86AudioRaiseVolume", hl.dsp.exec_cmd("dms ipc call audio increment 3"))
hl.bind(", XF86AudioLowerVolume", hl.dsp.exec_cmd("dms ipc call audio decrement 3"))
hl.bind(", XF86AudioMute",        hl.dsp.exec_cmd("dms ipc call audio mute"))
hl.bind(", XF86MonBrightnessUp",  hl.dsp.exec_cmd("dms ipc call brightness increment 5"))
hl.bind(", XF86MonBrightnessDown",hl.dsp.exec_cmd("dms ipc call brightness decrement 5"))
```

> **Verify with `dms ipc list`** — target/function names differ between DMS releases; this table comes from the official 1.4 docs + live `dms ipc list` on 1.5.3. DMS also has a built-in **hotkey overlay** (`Mod+Shift+/`) that shows the full cheatsheet.

**Common IPC targets:**

| Target | What | Examples |
|--------|------|----------|
| `audio` | Volume / mic | `audio setvolume 50`, `audio mute`, `audio cycleoutput` |
| `brightness` | Display brightness | `brightness set 80`, `brightness increment 10` |
| `clipboard` | Clipboard history | `clipboard toggle`, `clipboard clear` |
| `lock` | Lock screen | `lock lock` |
| `night` | Night mode | `night toggle`, `night temperature 3500`, `night schedule 20:00 06:00` |
| `notifications` | Notification center | `notifications toggle` |
| `processlist` | Task manager | `processlist focusOrToggle` |
| `screenshot` | Screenshots | `screenshot full`, `screenshot region` |
| `settings` | Settings panel | `settings focusOrToggle` |
| `spotlight` | Launcher | `spotlight toggle` |
| `wallpaper` | Wallpaper | `wallpaper set …`, `dankdash wallpaper` |
| `dankdash` | Dashboard | `dankdash wallpaper`, `dankdash toggle` |

---

## Application Theming (matugen)

DMS automatically generates theme files for native applications when matugen is enabled — on every wallpaper change and theme switch.

- **Disable entirely:** `DMS_DISABLE_MATUGEN=1` (or `true`) in the environment before launching DMS.
- **Custom templates:** add your own matugen templates in `~/.config/matugen/config.toml` — DMS executes them alongside its built-ins:

```toml
[config]

[templates.myapp]
input_path = '/home/<user>/.config/matugen/templates/myapp.conf'
output_path = '/home/<user>/.config/myapp/colors.conf'
```

- **dank16 palette** — DMS exposes the full Material-style palette to templates: `{{dank16.color0.default.hex}}`, `{{dank16.color0.dark.hex}}`, `{{dank16.color0.light.hex}}`. Terminal templates should use `.default` (DMS substitutes `.dark` for terminal configs when "Terminals — Always use Dark Theme" is on).

---

## Qt/KDE App Theming

DMS's own Qt UI + GTK apps follow the matugen Material theme via the `gtk3` platform theme (default env above). Standalone KDE apps (Dolphin, Kate, Gwenview) use the Qt fallback theme unless you give them a platform theme that reads KDE color schemes — verified on pop_arch (2026-08-19):

**Why not `QT_QPA_PLATFORMTHEME=kde`:** the `kde` platformtheme plugin ships with `plasma-integration`, which drags in krunner/libplasma/kscreenlocker. **`qt6ct-kde`** (AUR) is a patched `qt6ct` that understands KDE `.colors` files — exactly what matugen generates.

```bash
# 🔒 AUR — review PKGBUILD before installing
yay -S --noconfirm qt6ct-kde
```

**1. Point qt6ct at the matugen-generated KDE scheme** (DMS generates it via `matugenTemplateKcolorscheme` → `~/.local/share/color-schemes/DankMatugen*.colors`):

```ini
# ~/.config/qt6ct/qt6ct.conf
[Appearance]
custom_palette=true
color_scheme_path=/home/<user>/.local/share/color-schemes/DankMatugenDark.colors
```

**2. Set the platform theme to `qt6ct`** — the *package* is `qt6ct-kde`, but the *plugin/env value* is `qt6ct`:

```lua
hl.env("QT_QPA_PLATFORMTHEME", "qt6ct")
hl.env("QT_QPA_PLATFORMTHEME_QT6", "qt6ct")
```

**3. Enable DMS Qt theming** — Settings → toggle **Qt Theming** (`qtThemingEnabled: true` in `~/.config/DankMaterialShell/settings.json`). DMS's `scripts/qt.sh` writes the qt6ct config when it detects `/usr/bin/qt6ct` on PATH.

**Verify:** relaunch Dolphin — palette should match the active matugen scheme.

---

## Login Manager (SDDM / DankGreeter)

**Option A — SDDM (standard):** `sddm` from `extra`, plus `sddm` config + PAM hooks as in `Hyprland-Tiling.md` §SDDM. Works unchanged with Hyprland.

**Option B — DankGreeter (DMS-native):** `dms-greeter` handles seat management, D-Bus, and runtime dirs automatically — skip the manual systemd session setup in §3. Install + configure per the [DankGreeter docs](https://danklinux.com/docs/dankgreeter/installation) (mirrors the Niri guide's `dms greeter install` steps — see `Niri_Noctalia_v5.md` §greeter choice).

> Pick **one**. If you use DankGreeter, the `hyprland-session.target` manual setup above is unnecessary.

---

## Troubleshooting

- **DMS won't start** — `systemctl --user status dms`. Confirm the session target is running: `systemctl --user status hyprland-session.target`. The Lua hook must fire after `hyprland.start` — check `hyprctl log` for `dbus-update-activation-environment` errors.
- **`dms ipc` returns "not running"** — the shell is down. `systemctl --user restart dms`, or run `dms run` in a terminal and read the error.
- **Missing fonts / icons** — install `ttf-material-symbols-variable` (Material Symbols) + a Nerd Font, then `fc-cache -f`. Boxes instead of icons = missing Material Symbols.
- **Lua config errors** — `hyprctl` shows parse errors on launch; fix `hyprland.lua` before the session dies. Keep `require("dms.*")` files present (DMS generates them) or Hyprland won't boot.
- **DMS doesn't theme apps after wallpaper change** — check `dms ipc call doctor` (system diagnostics) and confirm `matugen` is on PATH.
- **faillock lock loop — sudo fails even with the correct password, greeter login bounces** (verified on pop_arch 2026-08-19): `sudo -A`/askpass is broken over SSH — journal shows `pam_unix(sudo:auth): conversation failed`. A background `yay --sudoloop --sudoflags "-A"` compounds it → `pam_faillock` locks the account → greeter login fails too (shared system-auth). **Fix:** `echo "<password>" | sudo -S <cmd>` instead of `-A`; never `yay --sudoloop -A`; reset via `su -c "faillock --reset --user <user>"` (su has no faillock chain).

---

## Uninstalling

```bash
# Remove DMS + Hyprland
sudo pacman -R --noconfirm dms-shell-hyprland hyprland

# Remove DMS configs
rm -rf ~/.config/DankMaterialShell ~/.config/hypr/dms ~/.local/state/DankMaterialShell ~/.cache/DankMaterialShell

# Remove the systemd session target binding
systemctl --user disable --now dms
rm -f ~/.config/systemd/user/hyprland-session.target
rm -f ~/.config/systemd/user/hyprland-session.target.wants/dms

# Remove optional deps if nothing else needs them
sudo pacman -R --noconfirm quickshell matugen dgop dsearch cava dankcalendar
```
