# Niri + Noctalia v5 🏔️🪽

Installing Niri (scrollable-tiling Wayland compositor) with Noctalia v5 on an existing Arch Linux + KDE Plasma + Plasma Login Manager setup.

> **v5 is beta.** This guide is based on the [official v5 docs](https://docs.noctalia.dev/v5/). v5 is a **native C++ rewrite** — completely different from v4 (Quickshell/QML). Package: `noctalia-git`, binary: `noctalia`, config: TOML under `[shell]`.

---

## 📖 Table of Contents

1. [Overview](#overview)
2. [Quick Reference — Keybinds & Usage](#quick-reference--keybinds--usage)
3. [Installation](#installation)
4. [Niri Configuration](#niri-configuration)
5. [Session File for Plasma Login Manager](#session-file-for-plasma-login-manager)
6. [Post-Install & Tweaks](#post-install--tweaks)
7. [Complete DE Experience](#complete-de-experience)
8. [Migration Notes (from KDE Plasma)](#migration-notes-from-kde-plasma)
9. [Uninstalling](#uninstalling)

---

## Overview

| Component | Package | Source | Purpose |
|-----------|---------|--------|---------|
| **Niri** | `niri` | `extra` (official) | Scrollable-tiling Wayland compositor |
| **Noctalia v5** | `noctalia-git` | AUR | Desktop shell: bars, launcher, dock, notifications, wallpaper, OSD, lock screen, clipboard, night light |
| **Plasma Login Manager** | `plasma-login-manager` | Already installed | Session switcher at login — keep KDE Plasma as alternative session |

**Key differences from Noctalia v4:**
- v5 is **native C++** (not Quickshell/QML)
- Package: `noctalia-git` (not `noctalia-shell`)
- Binary: `noctalia` (in PATH, not `/usr/local/bin/noctalia`)
- Config: TOML in `~/.config/noctalia/` under `[shell]` (not `[bar]`/`[dock]` top-level)
- Built-in: clipboard, night light, lock screen (no external `cliphist`/`wlsunset`/`swaylock` needed)

---

## Quick Reference — Keybinds & Usage

> **Mod** = <kbd>Super</kbd> (Windows key) on TTY, or <kbd>Alt</kbd> when running nested.

### 🪟 Window & Column Management

| Key | Action |
|-----|--------|
| <kbd>Mod</kbd> + <kbd>←</kbd>/<kbd>→</kbd> or <kbd>H</kbd>/<kbd>L</kbd> | Focus column left/right |
| <kbd>Mod</kbd> + <kbd>↑</kbd>/<kbd>↓</kbd> or <kbd>K</kbd>/<kbd>J</kbd> | Focus window up/down (within a column) |
| <kbd>Mod</kbd> + <kbd>Ctrl</kbd> + <kbd>←</kbd>/<kbd>→</kbd> | Move column left/right |
| <kbd>Mod</kbd> + <kbd>Ctrl</kbd> + <kbd>↑</kbd>/<kbd>↓</kbd> | Move window up/down (within a column) |
| <kbd>Mod</kbd> + <kbd>Q</kbd> | Close window |
| <kbd>Mod</kbd> + <kbd>F</kbd> | Maximize column (keeps gaps) |
| <kbd>Mod</kbd> + <kbd>Shift</kbd> + <kbd>F</kbd> | Fullscreen window |
| <kbd>Mod</kbd> + <kbd>V</kbd> | Toggle window floating |
| <kbd>Mod</kbd> + <kbd>R</kbd> | Cycle column width presets (1/3, 1/2, 2/3) |
| <kbd>Mod</kbd> + <kbd>=</kbd>/<kbd>-</kbd> | Widen/narrow column (±10%) |
| <kbd>Mod</kbd> + <kbd>W</kbd> | Toggle tabbed display (vertical tabs) |
| <kbd>Mod</kbd> + <kbd>[</kbd>/<kbd>]</kbd> | Consume/expel window into/from column |
| <kbd>Mod</kbd> + <kbd>C</kbd> | Center column on screen |

### 🖥️ Workspaces

| Key | Action |
|-----|--------|
| <kbd>Mod</kbd> + <kbd>1</kbd>–<kbd>9</kbd> | Switch to workspace N |
| <kbd>Mod</kbd> + <kbd>PageUp</kbd>/<kbd>PageDown</kbd> or <kbd>U</kbd>/<kbd>I</kbd> | Focus workspace up/down |
| <kbd>Mod</kbd> + <kbd>Ctrl</kbd> + <kbd>1</kbd>–<kbd>9</kbd> | Move column to workspace N |
| <kbd>Mod</kbd> + <kbd>Ctrl</kbd> + <kbd>PageUp</kbd>/<kbd>PageDown</kbd> | Move column to workspace up/down |
| <kbd>Mod</kbd> + <kbd>O</kbd> | Toggle Overview (zoom-out view) |
| <kbd>Mod</kbd> + <kbd>Scroll</kbd> | Scroll through workspaces |

### 🖥️ Multi-Monitor

| Key | Action |
|-----|--------|
| <kbd>Mod</kbd> + <kbd>Shift</kbd> + <kbd>←</kbd>/<kbd>→</kbd> | Focus monitor left/right |
| <kbd>Mod</kbd> + <kbd>Shift</kbd> + <kbd>Ctrl</kbd> + <kbd>←</kbd>/<kbd>→</kbd> | Move column to other monitor |

### 🏔️ Noctalia v5 IPC

| Key | Action |
|-----|--------|
| <kbd>Mod</kbd> + <kbd>Space</kbd> | Open launcher (app search, calculator, emoji) |
| <kbd>Mod</kbd> + <kbd>S</kbd> | Toggle control center |
| <kbd>Mod</kbd> + <kbd>,</kbd> | Toggle settings |
| <kbd>XF86AudioRaiseVolume</kbd> | Volume up (Noctalia OSD) |
| <kbd>XF86AudioLowerVolume</kbd> | Volume down (Noctalia OSD) |
| <kbd>XF86AudioMute</kbd> | Mute toggle |
| <kbd>XF86MonBrightnessUp</kbd> | Brightness up |
| <kbd>XF86MonBrightnessDown</kbd> | Brightness down |

### 📸 Screenshots

| Key | Action |
|-----|--------|
| <kbd>PrtSc</kbd> | Screenshot full screen |
| <kbd>Ctrl</kbd> + <kbd>PrtSc</kbd> | Screenshot current output |
| <kbd>Alt</kbd> + <kbd>PrtSc</kbd> | Screenshot focused window |

### 🔄 Other

| Key | Action |
|-----|--------|
| <kbd>Mod</kbd> + <kbd>Shift</kbd> + <kbd>E</kbd> | Quit Niri (shows confirmation) |
| <kbd>Mod</kbd> + <kbd>Shift</kbd> + <kbd>P</kbd> | Power off monitors |
| <kbd>Mod</kbd> + <kbd>Esc</kbd> | Toggle keyboard shortcuts inhibit |
| <kbd>Mod</kbd> + <kbd>Shift</kbd> + <kbd>/</kbd> | Show hotkey overlay |
| <kbd>Mod</kbd> + <kbd>T</kbd> | Launch terminal (default: `alacritty`) |

### 🐭 Mouse Gestures

| Gesture | Action |
|---------|--------|
| 4-finger swipe up | Toggle overview |
| Top-left hot corner | Toggle overview |
| <kbd>Mod</kbd> + scroll wheel | Switch workspaces |
| <kbd>Mod</kbd> + <kbd>Shift</kbd> + scroll wheel | Focus column left/right |

> **Tip:** Most actions can also be triggered via `niri msg action <action-name>`. See `niri msg action help` for the full list.
>
> **Default terminal:** Niri ships with `alacritty` as default. Konsole is already installed from your KDE setup — you can rebind `Mod+T` to Konsole if preferred.

---

## Installation

### 1. Install Niri

```bash
sudo pacman -S --noconfirm --needed niri
```

### 2. Install Recommended Packages

From the [Arch Wiki Niri page](https://wiki.archlinux.org/title/Niri#Installation), filtered for Noctalia v5 compatibility. Packages that Noctalia v5 already provides (launcher, bar, notifications, wallpaper, lock screen) are **excluded**.

**Core packages:**

```bash
sudo pacman -S --noconfirm --needed \
  alacritty \
  xdg-desktop-portal-gtk \
  xwayland-satellite \
  udiskie
```

| Package | Purpose | Why included |
|---------|---------|-------------|
| `alacritty` | Default terminal in Niri | Niri's `Mod+T` default — already installed on most setups |
| `xdg-desktop-portal-gtk` | Screen sharing (GTK) | Required for OBS, Chromium, Discord screenshare |
| `xwayland-satellite` | Run X11 applications | Some apps (e.g., older Electron apps) need XWayland |
| `udiskie` | Auto-mount USB drives | Convenience — tray icon for removable media |

**KDE users:** If you already have KDE Plasma, use `xdg-desktop-portal-kde` instead of `-gtk` for better integration:

```bash
sudo pacman -S --noconfirm --needed xdg-desktop-portal-kde
```

**Optional:**

```bash
# 🔒 AUR — Niri config GUI
yay -S --noconfirm --needed niri-settings-git
```

### 3. Install Noctalia v5

```bash
# 🔒 AUR — Review PKGBUILD before installing
yay -S --noconfirm --needed noctalia-git
```

This pulls in all native dependencies automatically. Noctalia v5 has **built-in** clipboard, night light, and lock screen — do NOT install `cliphist`, `wlsunset`, `swayidle`, or `swaylock` (they conflict with Noctalia's built-in functionality).

**Packages NOT needed (provided by Noctalia v5):**

| Replaced | Provided by Noctalia v5 |
|----------|------------------------|
| `fuzzel` (launcher) | Built-in launcher (`Mod+Space`) |
| `mako` (notifications) | Built-in notifications |
| `waybar` (bar) | Built-in bar |
| `swaybg` (wallpaper) | Built-in wallpaper engine |
| `swayidle` + `swaylock` (lock/idle) | Built-in lock screen + idle service |
| `cliphist` (clipboard) | Built-in clipboard manager |
| `wlsunset` (night light) | Built-in night light |
| `dms-shell-niri` / `noctalia-shell` | Replaced by `noctalia-git` (v5) |

---

## Niri Configuration

Create `~/.config/niri/config.kdl`:

```kdl
// ============================================================
// Niri + Noctalia v5 Config
// Based on: https://docs.noctalia.dev/v5/compositor-settings/niri/
// ============================================================

input {
    keyboard { xkb { layout "us,th" } }
}

// ---- Autostart Noctalia v5 ----
spawn-at-startup "noctalia"

// ---- Window Appearance ----
window-rule {
    geometry-corner-radius 20
    clip-to-geometry true
}

// Noctalia settings window: floating, fixed size
window-rule {
    match app-id="dev.noctalia.Noctalia"
    open-floating true
    default-column-width { fixed 1080; }
    default-window-height { fixed 920; }
}

debug {
    // Required for Noctalia notification actions & window activation
    honor-xdg-activation-with-invalid-serial
}

// ---- Wallpaper Backdrop ----
// Pick ONE option below. Uncomment your choice.

// Option 1: Blurred Overview (wallpaper visible only in overview, blurred)
// layer-rule {
//     match namespace="^noctalia-backdrop"
//     place-within-backdrop true
// }

// Option 2: Stationary Wallpaper (visible always, doesn't scroll) ★ RECOMMENDED
layer-rule {
    match namespace="^noctalia-wallpaper"
    place-within-backdrop true
}
layout {
    background-color "transparent"
}
overview {
    workspace-shadow { off }
}

// Option 3: Flat Color (no wallpaper, solid background)
// overview {
//     backdrop-color "#26233a"
// }

// ---- Blur ----
window-rule {
    background-effect { blur true xray false }
}
layer-rule {
    match namespace="^noctalia-(bar-[^"]+|notification|dock|panel|attached-panel|osd)$"
    background-effect { xray false }
}
blur {
    passes 2
    offset 3.0
    noise 0.03
    saturation 1.0
}

// ---- Keybinds ----
binds {
    // Noctalia IPC
    Mod+Space   { spawn-sh "noctalia msg panel-toggle launcher"; }
    Mod+S       { spawn-sh "noctalia msg panel-toggle control-center"; }
    Mod+Comma   { spawn-sh "noctalia msg settings-toggle"; }

    // Media keys (handled by Noctalia)
    XF86AudioRaiseVolume  { spawn-sh "noctalia msg volume-up"; }
    XF86AudioLowerVolume  { spawn-sh "noctalia msg volume-down"; }
    XF86AudioMute         { spawn-sh "noctalia msg volume-mute"; }
    XF86MonBrightnessUp   { spawn-sh "noctalia msg brightness-up"; }
    XF86MonBrightnessDown { spawn-sh "noctalia msg brightness-down"; }
}
```

### Wallpaper Options

Uncomment only **one** of the options above:

| Option | Layer Rule | Effect |
|--------|------------|--------|
| **Option 1** (Blurred Overview) | `match namespace="^noctalia-backdrop"` | Wallpaper visible only in overview, blurred & tinted |
| **Option 2** ★ (Stationary) | `match namespace="^noctalia-wallpaper"` | Wallpaper visible always, does not scroll with workspaces |
| **Option 3** (Flat Color) | `overview { backdrop-color "..." }` | Solid color, no wallpaper |

### Per-Widget Desktop Layer Rules

Noctalia v5 desktop widgets each have a unique namespace. Target individual widgets with `layer-rule`:

```kdl
// Target all desktop widgets
layer-rule {
    match namespace="^noctalia-desktop-widget-"
    // ... custom rules
}

// Target specific widget (e.g. weather)
layer-rule {
    match namespace="^noctalia-desktop-widget-weather-"
    // ... custom rules
}
```

Run `niri msg layers` to list all layer surfaces and see exact namespaces.

---

## Session File for Plasma Login Manager

### Create Wrapper Script

> **Important:** Use `niri-session` (not plain `niri`) — it exports environment variables to systemd correctly.

```bash
sudo tee /usr/local/bin/niri-noctalia-session << 'EOF'
#!/usr/bin/env bash
export XDG_CURRENT_DESKTOP=Noctalia
export XDG_SESSION_DESKTOP=noctalia
exec niri-session
EOF

sudo chmod +x /usr/local/bin/niri-noctalia-session
```

### Create Session `.desktop` File

```bash
sudo tee /usr/share/wayland-sessions/niri-noctalia.desktop << 'EOF'
[Desktop Entry]
Name=Niri + Noctalia v5
Comment=Session with Niri compositor and Noctalia v5 shell
Exec=/usr/local/bin/niri-noctalia-session
Type=Application
DesktopNames=Noctalia
EOF
```

### Test Noctalia First

Before rebooting, test that Noctalia starts under Niri:

```bash
# From a TTY (not inside Plasma)
niri-session -l
```

Or run Noctalia standalone for debugging:

```bash
noctalia
# Or daemon mode (returns after shell initialized):
noctalia --daemon
```

If it works, log out, select **"Niri + Noctalia v5"** from the Plasma Login Manager session dropdown, and log in.

---

## Post-Install & Tweaks

### Noctalia Config (TOML)

Noctalia v5 config lives at `~/.config/noctalia/config.toml`. The GUI settings editor is the primary way to configure, but here are useful manual overrides:

```toml
# ~/.config/noctalia/config.toml

[shell]
corner_radius_scale = 1.0        # 0 = square, 1 = default, 2 = extra rounded
clipboard_enabled = true          # built-in clipboard manager (no cliphist needed)
telemetry_enabled = false

[shell.panel]
transparency_mode = "solid"      # solid | soft | glass
borders = true
shadow = true

[shell.launcher]
categories = true
sort_by_usage = true

[hot_corners]
enabled = false
```

### Enable Type-to-Launch in Overview

```toml
# ~/.config/noctalia/config.toml
[shell]
niri_overview_type_to_launch_enabled = true
```

Or via GUI: **Settings → Niri → Overview**.

### Install Plugins

Noctalia v5 supports plugins. Browse and install via:
- GUI: **Settings → Plugins**
- Or manually from the [plugin registry](https://github.com/noctalia-dev/plugin-registry)

### Greeter (Login Screen)

Install Noctalia greeter as an alternative to Plasma Login Manager:

```bash
# 🔒 AUR — Review PKGBUILD before installing
yay -S --noconfirm --needed noctalia-greeter

# Then configure greetd to use it
# See: https://docs.noctalia.dev/v5/greeter/
```

> **Note:** This replaces plasma-login-manager as your display manager. Only do this if you want a unified Noctalia-themed login screen.

---

## Complete DE Experience

This section polishes Niri + Noctalia v5 into a full desktop environment. Config patterns adapted from [CachyOS Niri Noctalia](https://github.com/cachyos/cachyos-niri-noctalia) (translated from v4 to v5), with KDE-preferred packages.

### Additional Packages

> **If you already have KDE Plasma:** most of these are already installed — skip this section.

**KDE-preferred additions:**

```bash
# Core DE packages
sudo pacman -S --noconfirm --needed \
  dolphin \
  ark \
  gwenview \
  spectacle \
  kate \
  kdeconnect
```

| Package | Purpose |
|---------|---------|
| `dolphin` | GUI file manager (`Mod+E`) |
| `ark` | Archive manager (zip/tar/7z) |
| `gwenview` | Image viewer |
| `spectacle` | Screenshot tool |
| `kate` | Text editor |
| `kdeconnect` | Phone integration (notifications, file transfer, clipboard sync) |

**Cursor theme (optional):**

```bash
sudo pacman -S --noconfirm --needed capitaine-cursors
```

### Keybind Additions

Add these to the `binds {}` block in your niri config:

```kdl
    // ─── DE Applications ───
    Mod+Return               hotkey-overlay-title="Open Terminal" { spawn "alacritty"; }
    Mod+E                    hotkey-overlay-title="File Manager: Dolphin" { spawn "dolphin"; }
    Mod+B                    hotkey-overlay-title="Browser: Firefox" { spawn "firefox"; }
    Mod+Alt+L                hotkey-overlay-title="Lock" { spawn-sh "noctalia msg lock-screen"; }
    Mod+Shift+Q              hotkey-overlay-title="Session Menu" { spawn-sh "noctalia msg panel-toggle session"; }
```

> **Note:** `Mod+Return` replaces the default Niri `Mod+T` — matches the CachyOS convention.

### Input & Keyboard QoL

Add to `input {}` in your niri config:

```kdl
input {
    keyboard {
        xkb {
            layout "us,th"
        }
        numlock                        // Enable numlock on startup
    }
    touchpad {
        tap                            // Tap-to-click
        natural-scroll                 // Natural (macOS-style) scrolling
    }
    focus-follows-mouse                 // Focus windows under mouse pointer
    workspace-auto-back-and-forth       // Switch workspace back-and-forth
}
```

### Layout & Visual Polish

Add these to your niri config (or place in `layout {}`):

```kdl
layout {
    gaps 16                            // Gap between windows
    center-focused-column "never"      // Don't auto-center columns
    background-color "transparent"     // Required for Noctalia wallpaper
    preset-column-widths {
        proportion 0.33333
        proportion 0.5
        proportion 0.66667
    }
    struts {}
}
```

### Animations (Optional)

Add spring-physics animations for smooth workspace transitions:

```kdl
animations {
    workspace-switch {
        spring damping-ratio=1.0 stiffness=1000 epsilon=0.0001
    }
    horizontal-view-movement {
        spring damping-ratio=1.0 stiffness=900 epsilon=0.0001
    }
    window-open {
        duration-ms 200
        curve "ease-out-quad"
    }
    window-close {
        duration-ms 200
        curve "ease-out-cubic"
    }
    overview-open-close {
        spring damping-ratio=1.0 stiffness=900 epsilon=0.0001
    }
}
```

### Window Rules for Gaming

Steam popups and game launchers work better with floating rules:

```kdl
window-rule {
    match app-id="steam"
    exclude title=r#"^[Ss]team$"#
    open-floating true
}

window-rule {
    match app-id="steam" title=r#"^notificationtoasts_\d+_desktop$"#
    default-floating-position x=10 y=10 relative-to="bottom-right"
    open-focused false
}
```

### Modular Config Structure (Optional)

CachyOS splits niri config into `~/.config/niri/cfg/`. Replace your monolithic `config.kdl` with:

**`~/.config/niri/config.kdl`:**
```kdl
include "./cfg/animation.kdl"
include "./cfg/autostart.kdl"
include "./cfg/keybinds.kdl"
include "./cfg/input.kdl"
include "./cfg/display.kdl"
include "./cfg/layout.kdl"
include "./cfg/rules.kdl"
include "./cfg/misc.kdl"
```

Then split your config into these files. This makes it easier to manage — especially when moving between setups.

### Complete Environment Variables

Add to `cfg/misc.kdl` (or your main config):

```kdl
prefer-no-csd
screenshot-path null

environment {
    ELECTRON_OZONE_PLATFORM_HINT "auto"
    QT_QPA_PLATFORM "wayland"
    QT_QPA_PLATFORMTHEME "gtk3"
    QT_WAYLAND_DISABLE_WINDOWDECORATION "1"
    XDG_CURRENT_DESKTOP "niri"
    XDG_SESSION_TYPE "wayland"
}

cursor {
    xcursor-theme "capitaine-cursors"
    xcursor-size 24
}

hotkey-overlay {
    skip-at-startup
}
```

---

## Migration Notes (from KDE Plasma)

### What Stays

| Component | Status | Notes |
|-----------|--------|-------|
| **KDE Plasma** | 🟢 Available | Select "Plasma" session at login to go back |
| **Plasma Login Manager** | 🟢 Stays | Manages both sessions |
| **KDE apps** (Dolphin, Kate, Konsole) | 🟢 Work fine | Run under Wayland via Niri |
| **PipeWire** | 🟢 Already installed | Noctalia uses it for audio |
| **NetworkManager** | 🟢 Already installed | Noctalia reads NM state |
| **NVIDIA driver** | 🟢 Works | Niri supports NVIDIA via `nvidia-drm.modeset=1` |
| **Flatpak** | 🟢 Works | Requires `xdg-desktop-portal` (already installed) |

### What Changes

| Feature | KDE Plasma | Noctalia v5 |
|---------|-----------|-------------|
| Bar/Panel | Plasma Panel | Noctalia bar (configurable, per-monitor) |
| App Launcher | Kickoff | Noctalia launcher (`Mod+Space`) |
| Notifications | Plasma notifications | Noctalia notifications |
| Wallpaper | Plasma wallpaper | Noctalia wallpaper engine |
| System Tray | Plasma systray | Noctalia tray (Wayland protocol) |
| Lock Screen | KScreenLock | Noctalia lock screen |
| Clipboard | Klipper | Noctalia clipboard (built-in, no cliphist needed) |
| Night Light | KDE Night Color | Noctalia night light (built-in, no wlsunset needed) |
| Audio OSD | Plasma OSD | Noctalia OSD |

### NVIDIA Notes

If you have NVIDIA (from your existing guide):

- Ensure `nvidia-drm.modeset=1` is in your kernel cmdline (already set)
- Niri supports NVIDIA with the open driver via `nvidia-open-dkms`
- Add to your niri config if needed:
  ```kdl
  window-rule {
      match app-id=".*"
      render-element-callback-redraw true
  }
  ```

### Potential pitfalls

| Issue | Cause | Fix |
|-------|-------|-----|
| Noctalia not starting | Binary not found | v5 binary is `noctalia` (in PATH), not `/usr/local/bin/noctalia` |
| Config ignored | Using old v4 TOML format | v5 uses `[shell]` nested sections, not `[bar]`/`[dock]` |
| Clipboard not working | Missing `cliphist` | v5 has built-in clipboard — do NOT install `cliphist` |
| Night light not working | Missing `wlsunset` | v5 has built-in night light — do NOT install `wlsunset` |
| Screen share broken | xdg-desktop-portal not running | `systemctl --user enable --now xdg-desktop-portal` |
| Qt apps look wrong | Missing Qt Wayland support | Ensure `qt6-wayland` is installed (pulled by KDE) |
| Wallpaper wrong namespace | Using v4 namespace | Match `^noctalia-wallpaper` (Option 2) or `^noctalia-backdrop` (Option 1) |

---

## Uninstalling

If you want to remove Niri + Noctalia and go back to Plasma-only:

```bash
# Remove packages
sudo pacman -Rns --noconfirm niri
yay -Rns --noconfirm noctalia-git

# Remove session file
sudo rm /usr/share/wayland-sessions/niri-noctalia.desktop
sudo rm /usr/local/bin/niri-noctalia-session

# Remove configs
rm -rf ~/.config/niri
rm -rf ~/.config/noctalia

# Remove cache/state
rm -rf ~/.local/state/noctalia
rm -rf ~/.cache/noctalia

# Remove optional packages (if not needed by other things)
sudo pacman -Rns --noconfirm xwayland-satellite udiskie

# Verify Plasma login still works
systemctl status plasmalogin
```

---

*Noctalia v5 is beta — expect occasional updates. Keep packages up to date: `yay -Syu noctalia-git`.*
*Docs: https://docs.noctalia.dev/v5/* 🏔️🪽
*Arch Wiki: https://wiki.archlinux.org/title/Niri*
*llm-wiki: [[concepts/noctalia-v5]]*
