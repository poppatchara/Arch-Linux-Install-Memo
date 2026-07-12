# Niri + Noctalia v5 (Beta) 🏔️🪽

Installing Niri (scrollable-tiling Wayland compositor) with Noctalia v5 on an existing Arch Linux + KDE Plasma + Plasma Login Manager setup.

> **v5 is beta.** Stable users may prefer [v4 (Quickshell)](https://docs.noctalia.dev/v4/) via `noctalia-shell`. v5 is built on a native runtime (no Quickshell/QML) with a TOML-based config model.

---

## 📖 Table of Contents

1. [Overview](#overview)
2. [Quick Reference — Keybinds & Usage](#quick-reference--keybinds--usage)
3. [Installation](#installation)
4. [Niri Configuration](#niri-configuration)
5. [Session File for Plasma Login Manager](#session-file-for-plasma-login-manager)
6. [Post-Install & Tweaks](#post-install--tweaks)
7. [Migration Notes (from KDE Plasma)](#migration-notes-from-kde-plasma)
8. [Uninstalling](#uninstalling)

---

## Overview

| Component | Package | Source | Purpose |
|-----------|---------|--------|---------|
| **Niri** | `niri` | `extra` (official) | Scrollable-tiling Wayland compositor |
| **Noctalia v5** | `noctalia-git` | AUR (noctalia-git) | Desktop shell: bars, launcher, notifications, wallpaper, OSD, lock screen, dock |
| **Plasma Login Manager** | `plasma-login-manager` | Already installed | Session switcher at login — keep KDE Plasma as alternative session |

**Key differences from Noctalia v4:**
- v5 is **native C++** (not Quickshell/QML)
- Package: `noctalia-git` (not `noctalia-shell`)
- Binary: `/usr/local/bin/noctalia`
- Config: TOML files in `~/.config/noctalia/`
- IPC: `noctalia msg <command>`
- Required deps: wayland, cairo, pango, pipewire, wireplumber, etc.

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

### 🏔️ Noctalia v5 IPC (ต้องมีใน `binds {}`)

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
> **Default terminal:** Niri ships with `alacritty` as default. To change it, add a bind like `Mod+T { spawn "konsole"; }` in your config. (Konsole is already installed from your KDE setup.)

---

## Installation

### 1. Install Niri

```bash
sudo pacman -S niri
```

### 2. Install Noctalia v5

```bash
# v5 is AUR-only: noctalia-git
yay -S noctalia-git
```

This pulls in all native dependencies (meson, wayland-protocols, cairo, pango, libpipewire, wireplumber, polkit, etc.).

### 3. Install Optional Dependencies

```bash
# Night light feature
sudo pacman -S wlsunset

# Clipboard history (Noctalia built-in manager)
sudo pacman -S cliphist

# Audio visualizer support
sudo pacman -S cava
```

---

## Niri Configuration

Create `~/.config/niri/config.kdl`:

```kdl
// ============================================================
// Niri + Noctalia v5 Config
// Based on: https://docs.noctalia.dev/v5/getting-started/running-the-shell/
//           https://docs.noctalia.dev/v5/compositor-settings/niri
// ============================================================

input {
    keyboard { xkb { layout "us,th" } }
}

// ---- Autostart Noctalia v5 ----
spawn-at-startup "noctalia"

// ---- Window Appearance ----
window-rule {
    geometry-corner-radius 12
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

// ---- Blur ----
window-rule {
    background-effect { blur true xray false }
}
layer-rule {
    match namespace="^noctalia-(bar-[^\"]+|notification|dock|panel|attached-panel|osd)$"
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

    // Niri-native
    Mod+Shift+E { spawn "wlogout"; }
}
```

### Wallpaper Options

Uncomment only **one** of the options above:

| Option | Layer Rule | Effect |
|--------|------------|--------|
| **Option 1** (Blurred Overview) | `match namespace="^noctalia-backdrop"` | Wallpaper visible only in overview, blurred & tinted |
| **Option 2** ★ (Stationary) | `match namespace="^noctalia-wallpaper"` | Wallpaper visible always, does not scroll with workspaces |
| **Option 3** (Flat Color) | None — set `overview.backdrop-color` | Solid color, no wallpaper |

### Environment Variables

Noctalia v5 handles most env vars internally, but add these for Qt/Electron compatibility:

```kdl
environment {
    QT_QPA_PLATFORM "wayland"
    QT_QPA_PLATFORMTHEME "gtk3"
    ELECTRON_OZONE_PLATFORM_HINT "auto"
    GDK_BACKEND "wayland"
}
```

---

## Session File for Plasma Login Manager

### Create Wrapper Script

```bash
sudo tee /usr/local/bin/niri-noctalia-session << 'EOF'
#!/usr/bin/env bash
export XDG_CURRENT_DESKTOP=Noctalia
export XDG_SESSION_DESKTOP=noctalia
exec niri
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
niri --session noctalia
```

Or run Noctalia standalone for debugging:

```bash
noctalia
```

If it works, log out, select **"Niri + Noctalia v5"** from the Plasma Login Manager session dropdown, and log in.

---

## Post-Install & Tweaks

### Noctalia Config (TOML)

Noctalia v5 config lives at `~/.config/noctalia/config.toml`. The GUI settings editor is the primary way to configure, but here are useful manual overrides:

```toml
# ~/.config/noctalia/config.toml

[bar]
position = "top"
height = 40

[launcher]
enabled = true

[dock]
enabled = true
auto_hide = true

[night_light]
enabled = true
# Sunset/sunrise schedule (auto-detect location)
# or manual:
# schedule_start = "18:00"
# schedule_end = "06:00"
```

### Enable Type-to-Launch in Overview

```bash
# Option A: via TOML config
echo '[shell]
niri_overview_type_to_launch_enabled = true' >> ~/.config/noctalia/config.toml

# Option B: via GUI → Settings → Niri → Overview
```

### Install Plugins

Noctalia v5 supports **Luau plugins**. Browse and install via:
- GUI: **Settings → Plugins**
- Or manually from the [plugin registry](https://github.com/noctalia-dev/plugin-registry)

### Greeter (Login Screen)

Install Noctalia greeter as an alternative to Plasma Login Manager:

```bash
# AUR
yay -S noctalia-greeter

# Then configure greetd to use it
# See: https://docs.noctalia.dev/v5/greeter/
```

> **Note:** This replaces plasma-login-manager as your display manager. Only do this if you want a unified Noctalia-themed login screen.

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
| Clipboard | Klipper | Noctalia clipboard (uses cliphist) |
| Night Light | KDE Night Color | Noctalia night light (uses wlsunset) |
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
| Noctalia not starting | Missing deps | Run `noctalia` from terminal, check errors |
| Clipboard not working | cliphist not installed | `sudo pacman -S cliphist` |
| Night light not working | wlsunset not installed | `sudo pacman -S wlsunset` |
| Screen share broken | xdg-desktop-portal not running | `systemctl --user enable --now xdg-desktop-portal` |
| Qt apps look wrong | Missing env vars | Add `environment { QT_QPA_PLATFORM "wayland" }` to niri config |

---

## Uninstalling

If you want to remove Niri + Noctalia and go back to Plasma-only:

```bash
# Remove packages
sudo pacman -Rns niri
yay -Rns noctalia-git

# Remove session file
sudo rm /usr/share/wayland-sessions/niri-noctalia.desktop
sudo rm /usr/local/bin/niri-noctalia-session

# Remove configs
rm -rf ~/.config/niri
rm -rf ~/.config/noctalia

# Remove optional deps (if not needed by other things)
sudo pacman -Rns cliphist wlsunset

# Verify Plasma login still works
systemctl status plasmalogin
```

---

*Noctalia v5 is beta — expect occasional updates. Keep packages up to date: `yay -Syu noctalia-git`.*
*Docs: https://docs.noctalia.dev/v5/* 🏔️🪽
