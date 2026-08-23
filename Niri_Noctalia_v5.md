# Niri + Noctalia v5 🏔️🪽

Installing Niri (scrollable-tiling Wayland compositor) with Noctalia v5 (default) or DMS (secondary) on an existing Arch Linux install. **No Plasma/KWin required** — KDE apps (Dolphin, Kate, Gwenview) run standalone with `kded6` + a Qt platform theme (see [KDE Integration](#kde-integration) / [Qt/KDE App Theming](#qtkde-app-theming) for the two paths).

> **v5 is beta.** This guide is based on the [official v5 docs](https://docs.noctalia.dev/v5/). v5 is a **native C++ rewrite** — completely different from v4 (Quickshell/QML). Package: `noctalia-git`, binary: `noctalia`, config: TOML under `[shell]`.

---

## 📖 Table of Contents

1. [Overview](#overview)
2. [Quick Reference — Keybinds & Usage](#quick-reference--keybinds--usage)
3. [Installation](#installation)
4. [Niri Configuration](#niri-configuration)
5. [Session File for SDDM](#session-file-for-sddm)
6. [Post-Install & Tweaks](#post-install--tweaks)
7. [Complete DE Experience](#complete-de-experience)
8. [KDE Integration](#kde-integration)
9. [Login Manager (greeter choice)](#login-manager-greeter-choice)
10. [Migration Notes (from KDE Plasma)](#migration-notes-from-kde-plasma)
11. [Troubleshooting](#troubleshooting)
12. [DMS Shell Setup (secondary)](#dms-shell-setup-secondary)
13. [Uninstalling](#uninstalling)

---

## Overview

| Component | Package | Source | Purpose |
|-----------|---------|--------|---------|
| **Niri** | `niri` | `extra` (official) | Scrollable-tiling Wayland compositor |
| **Noctalia v5** | `noctalia-git` | AUR | Desktop shell: bars, launcher, dock, notifications, wallpaper, OSD, lock screen, clipboard, night light |
| **DMS** *(secondary)* | `dms-shell-niri` | `extra` (official) | DankMaterialShell — Quickshell+Go desktop shell: bar, spotlight launcher, notifications, control center, lock/idle, clipboard, wallpaper, night mode, theming |
| **Login manager** | `greetd` + greeter *or* `sddm` | `extra`/AUR | Display manager — **pick one**: builtin greeter (DankGreeter/Noctalia Greeter, lean) or SDDM + pixie (see [Login Manager (greeter choice)](#login-manager-greeter-choice)) |

**Pick one shell — Noctalia (default) or DMS (secondary).** Noctalia is a native C++ shell that replaces 6 separate tools (bar / launcher / notifications / wallpaper / lock+idle / clipboard / night light). DMS is the Quickshell+Go alternative — see [DMS Shell Setup (secondary)](#dms-shell-setup-secondary). Do **not** install both.

**Key differences from Noctalia v4:**
- v5 is **native C++** (not Quickshell/QML)
- Package: `noctalia-git` (not `noctalia-shell`)
- Binary: `noctalia` (in PATH, not `/usr/local/bin/noctalia`)
- Config: TOML in `~/.config/noctalia/` under `[shell]` (not `[bar]`/`[dock]` top-level)
- Built-in: clipboard, night light, lock screen (no external `cliphist`/`wlsunset`/`swaylock` needed)

---

## Quick Reference — Keybinds & Usage

> **Mod** = <kbd>Super</kbd> (Windows key) on TTY, or <kbd>Alt</kbd> when running nested.
>
> **niri 26.04 action/key names differ from older niri configs.** Verified working bindings:
> - `Mod+R` → `switch-preset-column-width` (older name `cycle-column-width` is gone)
> - No `widen-column`/`narrow-column` action exists in 26.04 — use `set-column-width <+/-10%>`
> - Workspace paging keys are `Page_Up`/`Page_Down` (not `PageUp`)
> - Terminal is `Mod+Return` (ghostty), **not** `Mod+T` (see Keybind Additions)
> - Rely on `niri validate` before applying; `niri msg action --help` lists the exact action names.

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
| <kbd>Mod</kbd> + <kbd>R</kbd> | Cycle column width presets (`switch-preset-column-width`) |
| <kbd>Mod</kbd> + <kbd>W</kbd> | Toggle tabbed display (vertical tabs) |
| <kbd>Mod</kbd> + <kbd>[</kbd>/<kbd>]</kbd> | Consume/expel window into/from column |
| <kbd>Mod</kbd> + <kbd>C</kbd> | Center column on screen |

### 🖥️ Workspaces

| Key | Action |
|-----|--------|
| <kbd>Mod</kbd> + <kbd>1</kbd>–<kbd>9</kbd> | Switch to workspace N |
| <kbd>Mod</kbd> + <kbd>Page_Up</kbd>/<kbd>Page_Down</kbd> | Focus workspace up/down (`focus-workspace-up`/`down`) |
| <kbd>Mod</kbd> + <kbd>Ctrl</kbd> + <kbd>1</kbd>–<kbd>9</kbd> | Move column to workspace N |
| <kbd>Mod</kbd> + <kbd>Ctrl</kbd> + <kbd>Page_Up</kbd>/<kbd>Page_Down</kbd> | Move column to workspace up/down |
| <kbd>Mod</kbd> + <kbd>Tab</kbd> | Toggle Overview (zoom-out view) |
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

### 🎛️ DMS IPC *(secondary shell only)*

> **Only if you chose DMS over Noctalia** — see [DMS Shell Setup (secondary)](#dms-shell-setup-secondary). Noctalia uses `noctalia msg ...` (table above); DMS uses `dms ipc call <target> <function>`. Run `dms ipc list` to enumerate available targets/functions while the shell is running.

| Key | DMS IPC command |
|-----|-----------------|
| <kbd>Mod</kbd> + <kbd>Space</kbd> | `dms ipc call spotlight toggle` (launcher) |
| <kbd>Mod</kbd> + <kbd>T</kbd> | Terminal (DMS default — `{{TERMINAL_COMMAND}}` → `ghostty`) |
| <kbd>Mod</kbd> + <kbd>V</kbd> | `dms ipc call clipboard toggle` (clipboard manager) |
| <kbd>Mod</kbd> + <kbd>M</kbd> | `dms ipc call mux toggle` (task manager — v1.5.3 target is `mux`, NOT `processlist`) |
| <kbd>Mod</kbd> + <kbd>,</kbd> | `dms ipc call settings focusOrToggle` (settings) |
| <kbd>Mod</kbd> + <kbd>N</kbd> | `dms ipc call notifications toggle` (notification center) |
| <kbd>Mod</kbd> + <kbd>Alt</kbd> + <kbd>N</kbd> | `dms ipc call night toggle` (night mode — see [Night Mode](#night-mode-built-in)) |
| <kbd>Mod</kbd> + <kbd>Y</kbd> | `dms ipc call dash toggle wallpaper` (wallpaper browser — `dankdash` deprecated in v1.5.3) |
| <kbd>Mod</kbd> + <kbd>Shift</kbd> + <kbd>/</kbd> | Hotkey overlay / keybind cheatsheet (DMS default) |
| <kbd>Mod</kbd> + <kbd>O</kbd> / <kbd>Mod</kbd> + <kbd>Tab</kbd> | Toggle overview (DMS default) |
| <kbd>Mod</kbd> + <kbd>P</kbd> | `dms ipc call wallpaper set <path>` (set wallpaper — see [Wallpaper & Layer Rules](#wallpaper--layer-rules)) |
| <kbd>XF86AudioRaiseVolume</kbd> | `dms ipc call audio increment 3` |
| <kbd>XF86AudioLowerVolume</kbd> | `dms ipc call audio decrement 3` |
| <kbd>XF86AudioMute</kbd> | `dms ipc call audio mute` |
| <kbd>XF86MonBrightnessUp</kbd> | `dms ipc call brightness list` → set (see [DMS Shell Setup](#dms-shell-setup-secondary)) |
| <kbd>XF86MonBrightnessDown</kbd> | same as above |

> **Verify function names on the live install** — `dms ipc list` is authoritative; DMS releases rename IPC functions between versions (the bind examples above come from the official docs).

### 📸 Screenshots

| Key | Action |
|-----|--------|
| <kbd>PrtSc</kbd> | Screenshot full screen |
| <kbd>Ctrl</kbd> + <kbd>PrtSc</kbd> | Screenshot current output |
| <kbd>Alt</kbd> + <kbd>PrtSc</kbd> | Screenshot focused window |
| <kbd>Mod</kbd> + <kbd>Shift</kbd> + <kbd>S</kbd> | Rectangle screenshot (Spectacle) |

### 🔄 Other

| Key | Action |
|-----|--------|
| <kbd>Left Alt</kbd> + <kbd>Left Shift</kbd> | Switch keyboard layout (`us` ↔ `th`) |
| <kbd>Mod</kbd> + <kbd>Shift</kbd> + <kbd>E</kbd> | Quit Niri (shows confirmation) |
| <kbd>Mod</kbd> + <kbd>Shift</kbd> + <kbd>P</kbd> | Power off monitors |
| <kbd>Mod</kbd> + <kbd>Esc</kbd> | Toggle keyboard shortcuts inhibit |
| <kbd>Mod</kbd> + <kbd>Shift</kbd> + <kbd>/</kbd> | Show hotkey overlay |
| <kbd>Mod</kbd> + <kbd>Return</kbd> | Launch terminal (default: `ghostty`) — **`Mod+T` is NOT bound on the Noctalia path** (CachyOS convention uses `Mod+Return`; see [Keybind Additions](#keybind-additions)). On the **DMS path**, `Mod+T` IS bound by DMS defaults (opens the same terminal) |

### 🐭 Mouse Gestures

| Gesture | Action |
|---------|--------|
| 4-finger swipe up | Toggle overview |
| Top-left hot corner | Toggle overview |
| <kbd>Mod</kbd> + scroll wheel | Switch workspaces |
| <kbd>Mod</kbd> + <kbd>Shift</kbd> + scroll wheel | Focus column left/right |

> **Tip:** Most actions can also be triggered via `niri msg action <action-name>`. See `niri msg action help` for the full list.
>
> **Default terminal:** Ghostty is the recommended terminal for this setup. Niri's factory default is alacritty, but we override to ghostty throughout this guide. Konsole is already installed from your KDE setup — you can use any terminal you prefer.

---

## Installation

### 1. Install Niri

```bash
sudo pacman -S --noconfirm --needed niri
```

> **Optional alternative:** `niri-qol-git` (fast-tracked QoL fork)
> *replaces* this package — see the Optional AUR section below.
> If you plan to use it, skip this step.

### 2. Install Recommended Packages

From the [Arch Wiki Niri page](https://wiki.archlinux.org/title/Niri#Installation), filtered for Noctalia v5 compatibility. Packages that Noctalia v5 already provides (launcher, bar, notifications, wallpaper, lock screen, clipboard, night light) are **excluded**.

> **Two Qt theme paths — pick per shell:** the **Noctalia path** (table below) uses `plasma-integration` + `kded6` as a single Qt theme layer (no `qt5ct`/`qt6ct`/`kvantum`, which fight each other). The **DMS path** (step 3b) needs only `kded6` + `qt6ct-kde` (AUR) and does **not** install `plasma-integration` — see [Qt/KDE App Theming](#qtkde-app-theming). Neither path needs `plasma-workspace` or `kwin` (verified on pop_arch 2026-08-19 — 19 Plasma/KWin packages removed, Dolphin/Kate still themed).

**Core packages:**

```bash
sudo pacman -S --noconfirm --needed \
  ghostty \
  libcanberra sound-theme-freedesktop \
  gnome-keyring \
  xdg-desktop-portal-kde \
  xdg-desktop-portal-gtk \
  xdg-utils \
  shared-mime-info \
  kde-cli-tools \
  plasma-integration \
  kded \
  xwayland-satellite \
  noto-fonts-emoji \
  wl-clipboard \
  adw-gtk-theme \
  polkit-kde-agent
```

| Package | Purpose | Why included |
|---------|---------|-------------|
| `ghostty` | Terminal emulator (`Mod+Return`) | Fast, native, feature-rich — replaces Niri's factory default (alacritty) |
| `libcanberra` | Sound event player | Freedesktop sound theme — plays feedback for volume, mute, notifications |
| `sound-theme-freedesktop` | Sound theme for `libcanberra` | The actual audio files `canberra-gtk-play -i audio-volume-change` plays — without this the volume binds are silent |
| `gnome-keyring` | Secret portal backend | Required for the Secret portal — password storage for GTK/Flatpak apps; PAM hook auto-starts it (see [Secret Storage](#secret-storage)) |
| `xdg-desktop-portal-kde` | Screen sharing (KDE) | **Noctalia path.** Preferred portal for a KDE-app setup — file picker, screencast. DMS path uses `xdg-desktop-portal-wlr` instead (see step 3b note) |
| `xdg-desktop-portal-gtk` | Screen sharing (fallback) | GTK portal backend for apps that don't use the KDE one |
| `xdg-utils` | MIME type & default apps | `xdg-open`, `xdg-mime` — without this Dolphin's "Open With" is empty |
| `shared-mime-info` | MIME type database | Freedesktop MIME database — file type detection |
| `kde-cli-tools` | KDE file associations | `keditfiletype`, `kioclient` — KDE app file type integration (optional on DMS path) |
| `plasma-integration` | Qt Platform Theme | **Noctalia path only.** Applies Breeze theme/colors/fonts to Qt apps when Plasma isn't running. DMS path uses `qt6ct-kde` (AUR) instead — see [Qt/KDE App Theming](#qtkde-app-theming) |
| `kded` | KDE daemon (`kded6`) | KDE file dialogs, mime types, trash — required for Dolphin/Kate to function fully (**both paths**) |
| `xwayland-satellite` | Run X11 applications | Some apps (e.g., older Electron apps) need XWayland; niri has no built-in XWayland |
| `noto-fonts-emoji` | Emoji fonts | Emoji rendering in terminal, browser, GTK apps |
| `wl-clipboard` | CLI clipboard | `wl-copy` / `wl-paste` — clipboard from terminal and scripts |
| `adw-gtk-theme` | GTK theme | Makes GTK apps (Firefox, VS Code) match the system look |
| `polkit-kde-agent` | GUI auth dialogs | **Noctalia path.** Required for any app needing root (partition managers, systemsettings) — auto-started in the Niri config. DMS path uses `polkit-gnome` (`/usr/lib/polkit-gnome/polkit-gnome-authentication-agent-1`) |

**Packages NOT needed (replaced by Noctalia v5 or the Qt6 path):**

| Replaced | Provided by |
|----------|-------------|
| `cliphist` | Noctalia v5 built-in clipboard |
| `fuzzel` / `wofi` | Noctalia v5 built-in launcher (`Mod+Space`) |
| `matugen` | Noctalia v5 wallpaper engine |
| `qt5-wayland` / `qt5ct` / `qt6ct-kde` / `kvantum` | `plasma-integration` (single Qt6 theme path) — **on the Noctalia path only**; `qt6ct-kde` becomes the Qt theme on the DMS path |
| `udiskie` | (dropped — no tray autostart wired in this config) |
| `gvfs-mtp` / `gvfs-gphoto2` | (optional — install only if you use MTP/camera import) |
| `accountsservice` | (auto-pulled by sddm if it needs it) |

**Optional — USB auto-mount (`udiskie`):** if you want removable-media mounting, install `udiskie` **and** add `spawn-at-startup "udiskie"` to the Niri config (it is not auto-started otherwise).

**Optional — MTP / camera import:** `sudo pacman -S --noconfirm --needed gvfs-mtp gvfs-gphoto2` for Android file transfer / digital camera import in Dolphin.

**Optional:**

```bash
# 🔒 AUR — NiriMod: visual, interactive config GUI
# Preserves manual settings & comments — better than niri-settings-git
# Repo: https://github.com/srinivasr/nirimod
yay -S --noconfirm --needed nirimod-git
```

**Optional — `niri-qol-git` (fast-tracked QoL fork):**

> 🔒 AUR — **soft-fork** of Niri that merges 3 upstream QoL PRs not yet
> in mainline. *Replaces* the base `niri` package (conflicts with
> `niri`/`niri-git`/`niri-bin`). If you install this, **skip the base
> `niri` install in the previous section** and use this instead.
> Safety-checked by Emi (2026-08-15: clean PKGBUILD, `--locked`/`--frozen`
> build, no network hooks — but 0 votes, brand-new package, so treat as
> experimental). Extra features:
> - **Sticky floating windows** — keep floating windows pinned across
>   workspaces (upstream PR #3302)
> - **Hidden workspaces** — hide workspaces without closing windows
>   (upstream PR #2997)
> - **`float-above-fullscreen`** window rule (upstream PR #4062)

```bash
yay -S --noconfirm --needed niri-qol-git
```

The `niri-qol` fork keeps the same `config.kdl` path
(`~/.config/niri/config.kdl`) and binary/commands as stock Niri — all
config in this guide applies unchanged. Built from source (Rust,
`--frozen`), so first install takes a while to compile.

### 3. Install Noctalia v5

```bash
# 🔒 AUR — Review PKGBUILD before installing
yay -S --noconfirm --needed noctalia-git
```

This pulls in all native dependencies automatically. Noctalia v5 has **built-in** clipboard, night light, and lock screen — do NOT install `cliphist`, `wlsunset`, `swayidle`, or `swaylock` (they conflict with Noctalia's built-in functionality).

### 3b. Install DMS (ALTERNATIVE — secondary, instead of step 3)

Skip this step unless you chose the DMS path — do **not** install both shells.

```bash
# DankMaterialShell — official extra (pulls dms-shell + dgop + quickshell + accountsservice)
sudo pacman -S --noconfirm --needed dms-shell-niri

# Required by DMS but NOT auto-pulled:
sudo pacman -S --noconfirm --needed \
  matugen \                        # color scheme engine — DMS runs the `matugen` binary at runtime
  ttf-material-symbols-variable \  # Material Symbols icon font (DMS icons/UI)
  ttf-jetbrains-mono-nerd          # Nerd Font for UI text
```

> **What this is:** [DankMaterialShell](https://github.com/AvengeMedia/DankMaterialShell) (DMS) is a **Quickshell + Go** desktop shell with Material 3 design — the Quickshell-based alternative to Noctalia's native C++ rewrite. `dms-shell-niri` is the Niri-specific extra package (depends on `dms-shell` + `niri`); a generic `dms-shell` variant exists for other compositors. Binary: `dms`, start with `dms run`, control with `dms ipc call <target> <function>`. Config lives in `~/.config/DankMaterialShell/`.
>
> **What DMS replaces:** bar, launcher (`dms ipc call spotlight toggle`), notifications, control center, lock screen + idle, clipboard manager, **wallpaper** (`dms ipc call wallpaper set ...`), and **night mode** (`dms ipc call night toggle` — gamma/color-temperature with suncalc scheduling). Theming is driven by **`matugen`** — this is why it's listed above: DMS spawns the `matugen` binary itself, and `dms-shell` does **not** declare it as a dependency (verified from `dms-shell`'s Depends On).
>
> **AUR policy:** `dms-shell-git` exists on AUR (development build) if you want newer than extra's `dms-shell` — review the PKGBUILD first (`yay -G dms-shell-git`). Extra's stable 1.5.x is the recommended choice.

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
| `noctalia-shell` | Replaced by `noctalia-git` (v5) |

> **DMS path note:** if you chose DMS (secondary) instead of Noctalia, the same "NOT needed" row applies to DMS differently — DMS covers bar / launcher / notifications / lock+idle / clipboard **and** wallpaper + night mode (built-in, verified from DMS source v1.5.3: `night` + `wallpaper` IPC targets with gamma/suncalc and wallpaper-set scheduling). Install `dms-shell-niri` **plus the required extras** in step 3b (notably `matugen`, which DMS execs but doesn't declare as a dependency) and skip step 3; do **not** install both shells.

---

### Plasma Integration (Optional)

> **DMS path: skip this.** Not needed — `plasma-workspace`/`plasma-browser-integration` were removed on pop_arch (19 Plasma/KWin packages dropped 2026-08-19) and the DMS desktop works without them. This whole subsection is the **Noctalia path's** optional extra for browser download notifications, media controls, and KDE Connect integration:

```bash
# plasma-workspace (~200MB disk, ~0MB RAM — nothing auto-starts)
# plasma-browser-integration (Firefox/Chromium extensions)
sudo pacman -S --noconfirm --needed \
  plasma-workspace \
  plasma-browser-integration
```

> **Note:** `plasmashell`/`kwin` don't run — only `kded6` and browser native messaging host. Install browser extensions.

#### KDE System Settings & GTK Theme Sync

```bash
# systemsettings : KDE config GUI
# kde-gtk-config : Sync KDE theme to GTK apps  
# breeze-gtk     : Breeze theme for GTK apps
sudo pacman -S --noconfirm --needed \
  systemsettings \
  kde-gtk-config \
  breeze-gtk
```

Disable Baloo if pulled: `balooctl disable`

## Niri Configuration

Create `~/.config/niri/config.kdl`:

```kdl
// ============================================================
// Niri + Noctalia v5 Config
// Based on: https://docs.noctalia.dev/v5/compositor-settings/niri/
// ============================================================

input {
    keyboard { xkb { layout "us,th" options "grp:lalt_lshift_toggle" } }
}

// ---- Autostart Noctalia v5 ----
spawn-at-startup "noctalia"
spawn-at-startup "kded6"   // KDE daemon — file dialogs, mime types, trash

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
    center-focused-column "never"      // Don't auto-center — prevents viewport shift on focus change
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
    Mod+Tab     { toggle-overview; }                       // Overview
    Mod+Shift+S { spawn "spectacle -r"; }                  // Rectangle screenshot

    // Audio & Brightness (with freedesktop sound feedback)
    XF86AudioRaiseVolume  { spawn-sh "noctalia msg volume-up; canberra-gtk-play -i audio-volume-change -d 'volume feedback'"; }
    XF86AudioLowerVolume  { spawn-sh "noctalia msg volume-down; canberra-gtk-play -i audio-volume-change -d 'volume feedback'"; }
    XF86AudioMute         { spawn-sh "noctalia msg volume-mute; canberra-gtk-play -i audio-volume-change -d 'volume feedback'"; }
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

## Session File for SDDM

### Create Wrapper Script

> **Critical — `XDG_RUNTIME_DIR` must be set explicitly.** sddm 0.21 does not
> always export `XDG_RUNTIME_DIR` into the Wayland session it launches. Without
> it, `niri --session` **panics with `RuntimeDirNotSet`** (`src/niri.rs:2435`),
> and the compositor never starts → the sddm login "bounces back" with a black
> screen. Every downstream error that looks like a GPU problem (`DRM atomic
> commit Permission denied`, `failed to add a framebuffer for the bo`,
> nvidia-open driver suspicion) is a **false signal** from the missing runtime
> env — not a driver bug. Always set it in the session wrapper.
>
> **`niri --session` vs `niri-session`:** upstream's shipped
> [`niri.desktop`](https://github.com/YaLTeR/niri/blob/main/resources/niri.desktop)
> uses `Exec=niri-session`, and the [Arch Wiki](https://wiki.archlinux.org/title/Niri)
> documents `niri-session` as the display-manager entry (it imports the login
> manager env via `systemd --user import-environment`, runs
> `dbus-update-activation-environment --all`, and starts `niri.service` → brings
> up `graphical-session.target`, which portals need). The direct `exec niri --session`
> is the **fallback**: on some sddm setups `niri-session` re-execs through a
> login shell and routes the compositor into `niri.service` detached with no
> controlling TTY → `error initializing the TTY backend`. **Try `exec niri-session`
> first** (keeps portal/graphical-session wiring); if the session black-screens
> or bounces, switch the wrapper's final line to `exec niri --session` (shown
> below — verified working on pop_arch).
>
> **Do NOT install `seatd`.** When a display manager (sddm) starts the session,
> systemd-logind grants the DRM seat automatically via the `pam_systemd` module
> in your `systemd-login`/`system-auth` stack; `seatd` is only for launching a
> compositor from a bare TTY with **no** display manager. Running `seatd` and
> logind together means two seat managers fighting over `/dev/dri/card*` — it
> does not help, and can break DRM master handoff on login.

```bash
sudo tee /usr/local/bin/niri-noctalia-session << 'EOF'
#!/usr/bin/env bash
# THE FIX: sddm 0.21 does not always export XDG_RUNTIME_DIR into the session.
# niri panics RuntimeDirNotSet without it. Set it explicitly (uid 1000 = pop).
export XDG_RUNTIME_DIR=/run/user/1000
export XDG_SESSION_TYPE=wayland
export XDG_CURRENT_DESKTOP=Noctalia
export XDG_SESSION_DESKTOP=noctalia
exec niri --session
EOF

sudo chmod +x /usr/local/bin/niri-noctalia-session
```

> **Debug when the session still bounces:** redirect niri's stderr so you can
> see the real panic/error the sddm launch produces (SSH-based tests of niri
> are misleading — they have no seat and always show "atomic denied"):
> change the last line to `exec niri --session 2>/tmp/niri_debug.log`, trigger a
> login, then read `/tmp/niri_debug.log`.

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
# From a bare TTY (outside the graphical session). niri-session -l is the TTY launch path —
# note it needs seatd/libseat for a bare-TTY seat, which is separate from the
# display-manager launch (logind grants the seat there). Don't confuse the two.
niri-session -l
```

Or run Noctalia standalone for debugging:

```bash
noctalia
# Or daemon mode (returns after shell initialized):
noctalia --daemon
```

If it works, log out, select **"Niri + Noctalia v5"** from your login manager's session dropdown (SDDM or greeter — see [Login Manager (greeter choice)](#login-manager-greeter-choice)), and log in.

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
| `ffmpegthumbs` | Video thumbnails KIO plugin | Pre-installed with KDE graphics |
| `ffmpegthumbnailer` | Video thumbnail generator | Install manually: `sudo pacman -S ffmpegthumbnailer` |

**Cursor theme (Phinger — matches the Hyprland guide):**

```bash
# Phinger — modern, clean cursor line (XCursor, AUR)
# 🔒 AUR — review `yay -G phinger-cursors` before installing
yay -S --noconfirm --needed phinger-cursors
```

> **⚠️ Theme name is NOT `phinger-cursors`** — the package installs four themes: `phinger-cursors-dark`, `phinger-cursors-light`, and `-left` variants. Use **`phinger-cursors-dark`** as the active theme in niri/GTK (see **Cursor Theme** under Niri Configuration). Setting `phinger-cursors` (no variant) renders nothing — verified on pop_arch 2026-08-19.

**Alternative:** Bibata Classic (AUR `bibata-cursor-git`) if you prefer the rounded Classic look — both work with Niri; just set the active theme in the cursors section above.

### Session Restore (Testing)

Niri has no built-in session restore. [nirinit](https://github.com/amaanq/nirinit) fills the gap — it auto-saves open windows every 5 minutes and restores them when Niri starts.

```bash
# 🔒 AUR — Review PKGBUILD before installing
yay -S nirinit-git
```

**Auto-start in Niri:**

```kdl
spawn-at-startup "nirinit"
```

State is saved to `~/.local/share/nirinit/session.json`. On login, nirinit reads it and re-launches all previously open apps on their correct workspaces. Window sizes and positions are preserved.

**Config (`~/.config/nirinit/config.toml`):**

```toml
# Map app_id to custom launch commands (PWAs, flatpaks, etc.)
[launch]
# "chromium-example.com__-Default" = "example-web-app"

# Apps to skip during restore
[skip]
# apps = ["steam"]
```

> ⚠️ `save_interval` is a **CLI flag only** (`--save-interval`), not a config field. Config only supports `[launch]` and `[skip]`. Using `save_interval` in config.toml causes: `unknown field 'save_interval', expected 'skip' or 'launch'`.

**Options:**
```
--save-interval 300   # Save interval in seconds (default: 300 = 5 min) — CLI flag only
--debug               # Verbose logging
```

**Important:** nirinit must be installed BEFORE starting Niri, or Niri must be restarted after install — `spawn-at-startup` only executes when the Niri session begins.

**Limitations:**
- Only restores which apps were open, not their content (browsers/editors handle their own tab restore)
- All restore commands go through Niri's `spawn`, so the app-id must be launchable via desktop entry

Alternative: [niri-session-manager](https://github.com/MTeaHead/niri-session-manager) — similar approach, auto-saves on exit and restores on login.

> ✅ **Working** — installed and tracking windows. Add to `spawn-at-startup` for full restore-on-login.

### PCoIP Remote Desktop (HP Anyware)

> **Full guide:** [PCoIP Keyboard Passthrough → `PCoIP_Niri_Workaround.md`](PCoIP_Niri_Workaround.md)
>
> PCoIP runs through a focus-based config swap (wrapper at `~/.local/bin/pcoip` swaps Niri config when the client gains/loses focus). The dedicated companion covers the problem, dual-config setup, nirimod compatibility, and verification.

### Run Command Dialog (Alt+F2 style)

Noctalia v5's built-in launcher (`Mod+Space`) already covers app search **and** "run command" (type a command, press Enter) — so no separate `wofi`/`fuzzel` is needed. To get a KDE-style *run* dialog, bind `Mod+F2` to the built-in launcher instead (see [Keybind Additions](#keybind-additions)):

### Keybind Additions

Add these to the `binds {}` block in your niri config. Multi-line blocks are used here for readability — note that **niri 26.04 also accepts inline one-line blocks** (`Mod+Space { spawn "..."; }` — verified with `niri validate` on 2026-08-19; DMS's own embedded binds use inline style), so either form deploys fine. `spawn-sh` goes on a separate line inside the block:

```kdl
    // ─── DE Applications (multiline for readability — inline also works on niri 26.04) ───
    Mod+Return {
        spawn "ghostty"
    }
    Mod+F2 {
        spawn-sh "noctalia msg panel-toggle launcher"
    }
    Mod+E {
        spawn "dolphin"
    }
    Mod+B {
        spawn "firefox"
    }
    Mod+Alt+L {
        spawn-sh "noctalia msg session lock"
    }
    Mod+Shift+Q {
        spawn-sh "noctalia msg panel-toggle session"
    }
    Mod+Tab {
        toggle-overview
    }
```

> **Note:** `Mod+Return` replaces the default Niri `Mod+T` — matches the CachyOS convention.

> **After editing, reload live (no logout needed):**
> ```bash
> NIRI_SOCKET=/run/user/1000/niri.wayland-<N>.<PID>.sock niri msg action load-config-file
> ```
> The socket path is `$XDG_RUNTIME_DIR/niri.wayland-<N>.<pid>.sock` — check the
> actual file in `/run/user/1000/`. Validate first with `niri validate`; reload
> with `niri msg action load-config-file` (the reload subcommand; older
> `reload-config` was renamed).\n
>
> **Gotchas (observed on pop_arch, niri 26.04):**
> - Missing binds cause "shortcut does nothing" — Mod+T terminal is **Mod+Return**
>   here; Mod+E/B/F2 are extra binds that MUST be added or they silently no-op.
>   A config that validates but lacks the binds block entries won't error.
> - Do NOT SSH into the same desktop user while diagnosing the live session:
>   the SSH session opens a second `user@UID` manager → logind tears down
>   `/run/user/<UID>` (socket files vanish) → every client gets ENOENT
>   ("not running") though the compositor process is alive and listening.
>   Debug via `NIRI_SOCKET=`/`XDG_RUNTIME_DIR=` env pointed at the live sockets
>   from a single short command instead.

### Input & Keyboard QoL

Add to `input {}` in your niri config:

```kdl
input {
    keyboard {
        xkb {
            layout "us,th"
            options "grp:lalt_lshift_toggle"
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

// Noctalia path (DMS path: QT_QPA_PLATFORMTHEME "gtk3" or "qt6ct" — see DMS section / Qt/KDE App Theming)
environment {
    ELECTRON_OZONE_PLATFORM_HINT "auto"
    QT_QPA_PLATFORM "wayland"
    QT_QPA_PLATFORMTHEME "kde"
    QT_QPA_PLATFORMTHEME_QT6 "kde"
    QT_STYLE_OVERRIDE "breeze"
    XDG_MENU_PREFIX "plasma-"
    QT_AUTO_SCREEN_SCALE_FACTOR "1"
    QT_ENABLE_HIGHDPI_SCALING "1"
    QT_SCALE_FACTOR_ROUNDING_POLICY "RoundPreferFloor"
    GTK_USE_PORTAL "1"
    QT_WAYLAND_DISABLE_WINDOWDECORATION "1"
    XDG_CURRENT_DESKTOP "niri"
    XDG_SESSION_TYPE "wayland"
    KDE_SESSION_VERSION "6"
    KDE_FULL_SESSION "true"
}

cursor {
    xcursor-theme "phinger-cursors-dark"
    xcursor-size 24
}

hotkey-overlay {
    skip-at-startup
}
```

> Also create `~/.config/environment.d/10-kde-on-niri.conf` with the same variables — see [KDE Integration](#kde-integration) for details on why both locations are needed.

---

## KDE Integration

KDE apps (Dolphin, Kate, Okular) run fine standalone, but they need background services for full functionality — file dialogs, theme consistency, trash support, network transparency.

### Required Services

> **Already covered — partially.** `kded` is part of the core package list above and is needed on **both paths** (KDE file dialogs/mime/trash). `plasma-integration` is the **Noctalia path's** Qt platform theme; on the DMS path it is NOT installed — `qt6ct-kde` handles Qt theming instead (see [Qt/KDE App Theming](#qtkde-app-theming)). This section explains *why* `kded6` matters and how to autostart it.

```bash
# If you skipped them earlier, install now (KDE file dialogs + Qt platform theme)
# Noctalia path:
sudo pacman -S --noconfirm --needed plasma-integration kded
# DMS path:
sudo pacman -S --noconfirm --needed kded
```

| Package | Provides | What breaks without it | Path |
|---------|----------|----------------------|------|
| `plasma-integration` | Qt Platform Theme | KDE apps use ugly fallback theme, wrong fonts & colors — **Noctalia path only** (DMS uses `qt6ct-kde`) | Noctalia |
| `kded` | KDE Daemon (`kded6`) | KDE file dialogs fail, mime types wrong, no trash support, `kioclient` broken | **Both** |

### Autostart KDE Daemon

Add to `~/.config/niri/config.kdl`:

```kdl
spawn-at-startup "kded6"
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

KDE apps (Dolphin network passwords, KDE Connect) need KWallet. GTK apps (VS Code, Chromium, Firefox) need GNOME Keyring via `libsecret`. Install both — they coexist:

```bash
sudo pacman -S --noconfirm --needed gnome-keyring libsecret kwallet kwalletmanager kwallet-pam

# PAM hooks for sddm — create the FULL sddm PAM stack, not just the hooks.
# WARNING: do NOT write only the 3 hook lines above (auth/session optional) —
# that yields a broken /etc/pam.d/sddm missing `auth include system-login`,
# so EVERY login fails with "Authentication failure" + "gkr-pam: no password
# is available". If sddm's PAM file doesn't exist yet (fresh install), build it
# complete; if it exists, append the optional hook lines to it.
#
# ⚠️ PAM CORRECTION (2026-08-19, verified on pop_arch): the session phase MUST
# keep `pam_systemd` — it lives in `system-login`, NOT `system-auth`. If you write
# `session include system-auth` (no system-login), niri 26.04 (TTY backend via
# smithay LibSeatSession/libseat) gets NO logind session → panics
# "error initializing the TTY backend ... Failed to open session: Function not
# implemented (os error 38)" → SDDM login bounces straight back to the greeter
# (sddm-helper exited with 101 = Rust panic). `system-login` carries pam_systemd
# which registers the session with logind.
if [ -f /etc/pam.d/sddm ] && ! grep -q 'system-login' /etc/pam.d/sddm; then
  echo "NOTE: existing /etc/pam.d/sddm lacks system-login include — check it"
fi
sudo tee /etc/pam.d/sddm <<'EOF'
#%PAM-1.0
auth        include     system-login
-auth       optional    pam_gnome_keyring.so
-auth       optional    pam_kwallet5.so

account     include     system-login

password    include     system-login
-password   optional    pam_gnome_keyring.so    use_authtok

session     optional    pam_keyinit.so          force revoke
session     include     system-login
-session    optional    pam_gnome_keyring.so    auto_start
-session    optional    pam_kwallet5.so         auto_start kwalletd=/usr/bin/ksecretd
EOF
```

> KWallet auto-unlock: wallet password = login password, blowfish encryption, wallet name = `kdewallet`.

### Dolphin "Open With" Blank Popup Fix

The most common issue: Dolphin shows an empty "Choose Application" dialog when opening files. This happens because KDE 6 renamed `applications.menu` to `plasma-applications.menu`, and `kbuildsycoca6` can't find it without `XDG_MENU_PREFIX=plasma-`.

**Root cause:** `kbuildsycoca6` — the KDE system configuration cache — is what populates Dolphin's "Open With" list. Without the menu file, it builds an empty database. The `plasma-applications.menu` file ships with `plasma-workspace` and lives at `/etc/xdg/menus/`.

**The fix requires three things:**

1. Set `XDG_MENU_PREFIX=plasma-` in your environment (so KDE looks for `plasma-applications.menu` instead of `applications.menu`)

2. Rebuild the KDE cache:
```bash
XDG_MENU_PREFIX=plasma- kbuildsycoca6 --noincremental
```

3. Auto-start `kded6` (the KDE daemon):
```kdl
spawn-at-startup "kded6"
```

**Environment variables must go in TWO places** — `~/.config/environment.d/` (so systemd/portals see them) AND niri's `environment {}` block (so niri-spawned apps see them). Systemd user services and portals can't read niri's `environment {}` block.

Create `~/.config/environment.d/10-kde-on-niri.conf`:

> **Important:** environment.d files use `KEY=VALUE` syntax and require **absolute paths**. `$HOME` and `%h` do NOT expand here.

```ini
# Noctalia path (DMS path: use QT_QPA_PLATFORMTHEME=qt6ct instead — see Qt/KDE App Theming)
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

| Variable | Why |
|----------|-----|
| `QT_QPA_PLATFORMTHEME=kde` | **Noctalia path** — use KDE's `plasma-integration` for theming (NOT `gtk3`/`qt6ct`) — lets `systemsettings` control Qt themes across Plasma and Niri. DMS path sets `qt6ct` instead (see [Qt/KDE App Theming](#qtkde-app-theming)) |
| `QT_STYLE_OVERRIDE=breeze` | **Noctalia path.** Forces Breeze widget style even when Plasma isn't running (DMS path doesn't set this) |
| `XDG_MENU_PREFIX=plasma-` | **Critical:** tells `kbuildsycoca6` to use `/etc/xdg/menus/plasma-applications.menu` — without this, Dolphin's "Open With" is empty |
| `GTK_USE_PORTAL=1` | Forces GTK apps to use the portal file picker (so they respect your KDE picker preference) |
| `XDG_DATA_DIRS` | Exposes Flatpak apps to KDE the same way Plasma does |

Sync the same variables in niri's `~/.config/niri/config.kdl`:

```kdl
environment {
    ELECTRON_OZONE_PLATFORM_HINT "auto"
    QT_QPA_PLATFORM "wayland"
    QT_QPA_PLATFORMTHEME "kde"
    QT_QPA_PLATFORMTHEME_QT6 "kde"
    QT_STYLE_OVERRIDE "breeze"
    XDG_MENU_PREFIX "plasma-"
    QT_AUTO_SCREEN_SCALE_FACTOR "1"
    QT_ENABLE_HIGHDPI_SCALING "1"
    QT_SCALE_FACTOR_ROUNDING_POLICY "RoundPreferFloor"
    GTK_USE_PORTAL "1"
    QT_WAYLAND_DISABLE_WINDOWDECORATION "1"
    XDG_CURRENT_DESKTOP "niri"
    XDG_SESSION_TYPE "wayland"
    KDE_SESSION_VERSION "6"
    KDE_FULL_SESSION "true"
}
```

Reload after creating: `systemctl --user daemon-reexec`

### KDE File Picker Instead of GNOME

Niri's default portal config uses `xdg-desktop-portal-gnome` for file pickers (which means Nautilus). To force the KDE file picker instead, create `~/.config/xdg-desktop-portal/niri-portals.conf`:

```ini
[preferred]
default=kde;gnome;gtk;
org.freedesktop.impl.portal.FileChooser=kde;
```

Keep `xdg-desktop-portal-gnome` installed — it's still needed for screencasting. `default=kde;gnome;gtk;` prioritizes KDE portals for everything, falling back to GNOME then GTK.

**Firefox:** In `about:config`, set `widget.use-xdg-desktop-portal.file-picker` to `1` to force it through the portal.

### kded6 + Noctalia Tray Conflict

If your system tray disappears after starting `kded6`, it's because kded6 registers its own `StatusNotifierWatcher`, overriding Quickshell/Noctalia's. Add this BEFORE `kded6` in autostart:

```kdl
spawn-at-startup "dbus-update-activation-environment" "--systemd" "--all"
spawn-at-startup "kded6"
```

### Prevent KDE File Picker Fullscreen

Without Plasma, the KDE file picker can open fullscreen. Add a window rule to constrain it:

```kdl
window-rule {
    match app-id="org.freedesktop.impl.portal.desktop.kde"
    default-column-width { proportion 0.5; }
    default-window-height { proportion 0.8; }
    open-floating true
    open-fullscreen false
}
```

### Dolphin Dialogs as Floating Popups

Without Plasma, Dolphin's delete confirmation, copy progress, and other operation dialogs open as tiled windows instead of popups. Match them by title pattern:

```kdl
window-rule {
    match app-id="org.kde.dolphin" title="^(Delete|Deleting|Copy|Copying|Mov|Moving|Renaming|Creating) "
    open-floating true
    default-column-width { fixed 500; }
    default-window-height { fixed 250; }
}
```

To find app-ids and titles for other windows, use `niri msg pick-window` then click the window.

### Video Thumbnails in Dolphin

```bash
sudo pacman -S --noconfirm --needed ffmpegthumbnailer
```

(`ffmpegthumbs` is already installed by default with KDE graphics packages.)

> See [Resources](#resources) at the bottom for all references and reading material.

---

## Login Manager (greeter choice)

Both Noctalia and DMS have a **built-in greetd-based greeter** — the lean option. SDDM + pixie remains available for a KDE-style login. **Pick ONE:**

| Option | Shell path | Packages added | Setup | Look |
|--------|-----------|----------------|-------|------|
| **A — Builtin greeter** (recommended, lean) | DMS → **DankGreeter** (`greetd-dms-greeter-bin`); Noctalia → **Noctalia Greeter** (`noctalia-greeter`) | `greetd` + greeter (DankGreeter adds **only `quickshell`** — already present from `dms-shell`; Noctalia Greeter bundles its own wlroots compositor) | `dms greeter install` (DMS) / config drop-in (Noctalia) | Matches the shell's Material/noctalia theme automatically |
| **B — SDDM + pixie** | Both | `sddm` + `kwin` (65 deps — whole Plasma stack) + `layer-shell-qt` + `pixie-sddm-git` | Manual: PAM rewrite + Wayland greeter config + session wrapper | KDE Material-style (two-tone clock, blur, avatar) |

> **Why Option A is lean:** sddm's Wayland greeter runs **`kwin_wayland`** as its compositor, and `kwin` pulls in ~35 KDE/Plasma packages (aurorae, breeze, plasma-activities, kauth, kcmutils, kcolorscheme, kconfig, kcoreaddons, kcrash, kdbusaddons, kdeclarative, kdecoration, kglobalaccel, kpackage, ksvg, kwayland, …) **plus** `xorg-server`/`xorg-xauth` (sddm deps even on the Wayland path). DankGreeter runs on **Quickshell** — already a `dms-shell` dependency, so switching to it adds exactly **one** package (`greetd`). No PAM rewriting, no session wrapper, no X11. Verified dependency counts 2026-08-19 on pop_arch.
>
> **Noctalia Greeter vs DankGreeter:** `noctalia-greeter` (AUR, maintainer `noctalia-dev` = the shell's author, 1.2.1) bundles a wlroots compositor and greets with the noctalia Material theme; use it on the Noctalia path. `greetd-dms-greeter-bin` (AUR, 1.5.2, deps `greetd` + `quickshell`) is DMS's own greeter — the DMS docs call it **DankGreeter** (`dms greeter install` automates the whole setup). **Do not install both greeters.**
>
> **When SDDM still makes sense (Option B):** you already have Plasma installed (kwin etc. are already present as deps), or you want the KDE login-theme ecosystem and the same greeter as your Hyprland machines.

### Option A — Builtin greeter (DankGreeter / Noctalia Greeter)

**DMS path (DankGreeter)** — install + auto-configure:

```bash
# 1. greetd (minimal display manager) + DMS's greeter (deps: greetd + quickshell — already installed)
yay -S --noconfirm --needed greetd-dms-greeter-bin

# 2. One command replaces your current display manager with the greeter (installs/updates
#    /etc/greetd/config.toml, creates the greeter user, enables greetd.service, disables sddm)
dms greeter install
```

> `dms greeter` subcommands: `install` (install + configure + switch DM), `enable`/`disable` (toggle in greetd config), `status` (check sync), `sync` (push DMS theme/settings to the greeter), `uninstall` (restore previous display manager). Greeter system user + tmpfiles are shipped by the package (`/usr/lib/sysusers.d/dms-greeter.conf`, `/usr/lib/tmpfiles.d/dms-greeter.conf`). The greeter UI matches your DMS theme automatically via `dms greeter sync`.

**Noctalia path (Noctalia Greeter)**:

```bash
# 1. greetd + Noctalia's greeter (bundles a wlroots compositor; maintainer = noctalia-dev)
yay -S --noconfirm --needed greetd noctalia-greeter

# 2. Point greetd at it
sudo tee /etc/greetd/config.toml <<'EOF'
[terminal]
vt = 1

[default_session]
command = "/usr/bin/noctalia-greeter-session"
user = "greeter"
EOF

# 3. Enable greetd (replaces sddm)
sudo systemctl enable --now greetd
sudo systemctl disable sddm 2>/dev/null || true
```

> Greeter user needs seat access: `sudo usermod -aG video,seat greeter` (the Noctalia Greeter package may do this itself — verify with `getent group greeter`). Do **not** install `seatd` — `systemd-logind` already provides seat management; `seatd` conflicts with it.
>
> **Session files work the same** — greetd lists `/usr/share/wayland-sessions/*.desktop`, so "Niri + Noctalia"/"Niri + DMS" appear automatically (no wrapper needed on the greetd path; the compositor runs directly on the VT).

### Option B — SDDM + pixie theme

> Skip to [Migration Notes](#migration-notes-from-kde-plasma) if you chose Option A. Everything below is the SDDM path (heavier, KDE-style login).

### 1. Install SDDM + pixie theme

```bash
# SDDM (official repo) — must be >= 0.20.0 to avoid bug #1476 (90s shutdowns).
# Arch's package is current (0.21+), so this is already satisfied.
sudo pacman -S --noconfirm --needed sddm

# Qt6 engine prereqs (SDDM on Arch runs Qt6) — the pixie PKGBUILD lists Qt5 deps,
# but missing these on Qt6 SDDM = black screen.
sudo pacman -S --noconfirm --needed qt6-declarative qt6-svg

# pixie theme (AUR, maintained by the theme author xCaptaiN09)
# 🔒 AUR — review `yay -G pixie-sddm-git` before installing if desired.
yay -S --noconfirm --needed pixie-sddm-git
```

### 2. Point SDDM at pixie

```bash
sudo sh -c 'echo "\
[Theme]\
Current=pixie" > /etc/sddm.conf.d/theme.conf'
```

> **Why the drop-in dir:** `/etc/sddm.conf.d/` overrides the packaged defaults without touching `/etc/sddm.conf` — survives package updates.
>
> **AUR policy:** `pixie-sddm-git` installs only to `/usr/share/sddm/themes/pixie` (Main.qml, components, assets, theme.conf). Verify with `yay -G pixie-sddm-git` if desired.
>
> **Other themes:** any SDDM theme from the [KDE Store](https://store.kde.org/browse/cat/106/order/latest/) or AUR (`sddm-theme-*`) works — just change `Current=` accordingly.

### 2b. Use the Wayland greeter (drops the Xorg process — saves ~135MB RAM)

SDDM defaults to an **X11 greeter** (`/usr/lib/Xorg` + `sddm-greeter-qt6` on VT1). On a Niri/Wayland machine that Xorg serves **only the login screen** — after login it is dead weight and, worse, can be left running (stuck on VT1) if the session switches VT, wasting ~135MB forever.

Switch SDDM to a **Wayland greeter** so the greeter runs inside a Wayland compositor and is torn down cleanly with the session. Requires the `kwin` package (has `kwin_wayland`) — present if Plasma is installed, or `sudo pacman -S kwin` — plus `layer-shell-qt` (lets the QML theme render as a `wlr_layer_shell` surface):

```bash
sudo pacman -S --needed kwin layer-shell-qt

sudo tee /etc/sddm.conf.d/01-wayland.conf > /dev/null <<'EOF'
[General]
DisplayServer=wayland
GreeterEnvironment=QT_WAYLAND_SHELL_INTEGRATION=layer-shell,XCURSOR_THEME=phinger-cursors-dark

[Wayland]
CompositorCommand=kwin_wayland --drm --no-lockscreen --no-global-shortcuts --locale1
EOF
sudo systemctl restart sddm
```

> **Config is from [Arch Wiki — SDDM §Wayland → KDE Plasma/KWin](https://wiki.archlinux.org/title/SDDM#KDE_Plasma_/_KWin).** `CompositorCommand` needs the full flag set (`--drm` selects KMS/DRM rendering, `--no-global-shortcuts` drops the KWin shortcuts layer that has no meaning on a login screen, `--locale1` reads the keyboard layout from `localectl`); `--no-lockscreen` alone is not enough. `GreeterEnvironment=QT_WAYLAND_SHELL_INTEGRATION=layer-shell` is what makes a QML theme (pixie, breeze) composite as an actual layer-shell surface — without it the greeter window may not render.
>
> **`XCURSOR_THEME` is required, not optional:** sddm injects `GreeterEnvironment` verbatim into the greeter, and its packaged default leaves `XCURSOR_THEME` empty — which makes the Wayland greeter show **no cursor at all**. Set it to your active cursor theme here (e.g. `phinger-cursors-dark`); do **not** try to force it via `~/.config` X resources or PAM `pam_env` inside the greeter, which are ignored on the Wayland greeter path.
>
> **Prereq for the Wayland greeter:** the session wrapper **must** export `XDG_RUNTIME_DIR` (see the [Wrapper Script](#create-wrapper-script) fix above). Without it SDDM's Wayland-path session launch fails with `RuntimeDirNotSet` and the login "bounces back" — that is why the greeter was reverted to X11 before; it is now safe because the wrapper sets `XDG_RUNTIME_DIR=/run/user/1000`.
>
> **`uwsm` is NOT needed here.** [Hyprland wiki — Systemd startup](https://wiki.hypr.land/Useful-Utilities/Systemd-start/) describes `uwsm` only for launching a compositor **from a tty/console** (it generates a `hyprland-uwsm.desktop`). When SDDM is the display manager it lists the session and starts it with its own PAM-integrated session handling — no `uwsm` involved.
>
> **Verify the greeter is Wayland (no Xorg):** after restart, `pgrep -x Xorg` returns nothing; `pgrep -f kwin_wayland` shows the greeter compositor. Revert to the X11 greeter with `rm /etc/sddm.conf.d/01-wayland.conf && sudo systemctl restart sddm`.

### 3. Verify the session entry

```bash
# List available sessions — a "Niri + Noctalia" entry must be present
ls /usr/share/wayland-sessions/
ls /usr/share/xsessions/
```

The Niri session file is created in the [Session File for SDDM](#session-file-for-sddm) section above; SDDM lists every `*.desktop` it finds, so "Niri + Noctalia" (and Plasma, if installed) will appear.

### 4. Test then enable SDDM

**Do NOT disable Plasma Login Manager yet.** Test first:

```bash
# Start SDDM on a separate VT for testing (leave your current session running)
sudo systemctl start sddm
```

Switch to TTY1 (SDDM's default VT) — you should see the pixie login screen with the "Niri + Noctalia" session entry. If it works, switch back with `Ctrl+Alt+F2`.

If SDDM doesn't appear, check logs:

```bash
journalctl -u sddm -f
```

Once tested:

```bash
# Enable SDDM
sudo systemctl enable sddm --now

# Disable Plasma Login Manager (if it was enabled)
sudo systemctl disable plasmalogin

# Reboot
sudo reboot
```

After reboot, SDDM shows the pixie login screen — select **"Niri + Noctalia"**.

### 5. Remove the Plasma session packages (Option B — lean)

Once Niri + Noctalia is stable and you no longer need the Plasma *session* fallback, remove the desktop shell and the old login manager (which forces `plasma-workspace`):

```bash
# -Rns removes unused deps that only these pulled in.
# plasma-desktop + kdeplasma-addons = the Plasma session (plasmashell) you won't log into.
# plasma-login-manager  = the DM that depends on plasma-workspace.
sudo pacman -Rns --noconfirm plasma-desktop kdeplasma-addons plasma-login-manager

# systemctl will complain that plasmalogin.service no longer exists — confirm sddm is enabled:
systemctl is-enabled sddm
```

> **Keep `kded`** — the KDE daemon provides the file dialogs, mime types, and trash that Dolphin/Kate rely on (it's not the Plasma session; it doesn't auto-start plasmashell/kwin). **`plasma-workspace` and `plasma-integration` can be removed** on the DMS path (verified on pop_arch 2026-08-19 — 19 Plasma/KWin packages dropped: `sddm kwin plasma-workspace plasma-integration plasma-browser-integration kscreenlocker layer-shell-qt polkit-kde-agent kde-cli-tools kde-gtk-config systemsettings kwalletmanager kwallet-pam aurorae breeze breeze-gtk qqc2-breeze-style xdg-desktop-portal-kde pixie-sddm-git`). Qt theming then comes from `qt6ct-kde` instead (see [Qt/KDE App Theming](#qtkde-app-theming)). Note `xdg-desktop-portal-kde` removal → install `xdg-desktop-portal-wlr` (niri/lutris need a `xdg-desktop-portal-impl` provider) and swap `polkit-kde-agent` → `polkit-gnome`.

### Rollback

If anything goes wrong:

```bash
# From a TTY (Ctrl+Alt+F3), log in and:
# To go back to SDDM's X11 greeter (drop the Wayland greeter config):
sudo rm /etc/sddm.conf.d/01-wayland.conf && sudo systemctl restart sddm

# To restore the full Plasma session fallback (reinstalls the DM + desktop you removed):
sudo pacman -S --noconfirm plasma-login-manager plasma-desktop
sudo systemctl disable sddm
sudo systemctl enable plasmalogin
sudo reboot
```

## Migration Notes (from KDE Plasma)

### What Stays

| Component | Status | Notes |
|-----------|--------|-------|
| **KDE Plasma (session)** | 🔴 Removed | `plasma-desktop`/`kdeplasma-addons` dropped — no Plasma session fallback |
| **KDE app services** (`plasma-workspace`/`plasma-integration`/`kded`) | 🟢 Stays | Keeps Dolphin/Kate file dialogs, trash, Qt6 Breeze theming |
| **SDDM** | 🟢 Stays | Now the sole display manager |
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
| Qt apps look wrong | Missing Qt Wayland support or theme path | Noctalia path: ensure `qt6-wayland` (pulled by `plasma-integration`) + `plasma-integration`/`kded6` installed + autostarted (see [Required Services](#required-services)). DMS path: ensure `qt6ct-kde` + `QT_QPA_PLATFORMTHEME=qt6ct` (see [Qt/KDE App Theming](#qtkde-app-theming)) |
| Dolphin "Open With" empty | Missing `xdg-utils` + `kde-cli-tools` | `sudo pacman -S --noconfirm --needed xdg-utils shared-mime-info kde-cli-tools`, then set defaults with `xdg-mime` (see "Dolphin Open With Blank Popup Fix" in KDE Integration) |
| Wallpaper wrong namespace | Using v4 namespace | Match `^noctalia-wallpaper` (Option 2) or `^noctalia-backdrop` (Option 1) |

---

## Troubleshooting

### polkit Authentication Agent

Some applications (grub-customizer, systemsettings, etc.) need a polkit authentication dialog. Without a polkit agent, these apps either fail silently or show a text prompt in the terminal.

**Noctalia path** (KDE stack present):

```bash
sudo pacman -S --noconfirm --needed polkit-kde-agent
```

```kdl
spawn-at-startup "/usr/lib/polkit-kde-authentication-agent-1"
```

**DMS path** (KDE stack removed — `polkit-kde-agent` was part of the KDE/Plasma dependency chain and got removed with it):

```bash
sudo pacman -S --noconfirm --needed polkit-gnome
```

```kdl
spawn-at-startup "/usr/lib/polkit-gnome/polkit-gnome-authentication-agent-1"
```

> **⚠️ Portal impl on the DMS path:** `niri` and `lutris` depend on `xdg-desktop-portal-impl`. The KDE path satisfies it with `xdg-desktop-portal-kde`; on the DMS path (no KDE stack) install the lightweight wlroots backend instead:
>
> ```bash
> sudo pacman -S --noconfirm --needed xdg-desktop-portal-wlr
> ```
>
> Verified on pop_arch 2026-08-19 — `xdg-desktop-portal-wlr` 0.8.2 satisfies `niri` + `lutris` after removing `xdg-desktop-portal-kde`.

### DISPLAY=:0 for X11 Apps

Minimal Wayland compositors like Niri do not set `DISPLAY`. Some X11 apps (including pcoip-client via XWayland) need this variable to find the XWayland display.

Create `~/.config/environment.d/display.conf`:

```
DISPLAY=:0
```

And add to `~/.bashrc`:

```bash
export DISPLAY=:0
```

> **Note:** `environment.d` files use `KEY=VALUE` syntax and require **absolute paths**. They are read by systemd user services and portals on login.

---
## DMS Shell Setup (secondary)

> **Alternative path — only if you chose DMS over Noctalia.** Skip this entire section otherwise.

[DankMaterialShell](https://github.com/AvengeMedia/DankMaterialShell) (DMS) is a **Quickshell + Go** desktop shell — the Material-3-styled alternative to Noctalia's native C++ rewrite. It replaces bar, launcher (spotlight), notifications, control center, lock screen + idle, clipboard, wallpaper, and **night mode** (built-in gamma/color-temperature) — all verified from DMS source v1.5.3.

### Niri Config Integration

DMS ships niri-ready config fragments. Include them at the end of `~/.config/niri/config.kdl` (create the files first):

```bash
mkdir -p ~/.config/niri/dms
touch ~/.config/niri/dms/{colors,layout,alttab}.kdl
# ⚠️ Do NOT touch binds.kdl empty — see "Keybinds" below. It must contain DMS's default binds.
```

```kdl
// ~/.config/niri/config.kdl — append at the end
// DMS managed fragments: colors (theme), layout (gaps/radius), alttab (switcher), binds (DMS IPC)
include "dms/colors.kdl"
include "dms/layout.kdl"
include "dms/alttab.kdl"
include "dms/binds.kdl"
```

> **Files must exist before the include lines** — niri fails to parse a config that includes a missing file. If a fragment is not wanted, delete its `include` line instead of leaving a dangling file.
>
> **⚠️ `dms/binds.kdl` must contain DMS's default binds** — do NOT create it empty with `touch` (that silently disables all DMS default keybinds: window management, workspace switching, media keys, screenshots, etc. — verified on pop_arch 2026-08-19: only guide-added binds worked, all 120+ DMS defaults were gone). DMS creates/updates this file when you open **Settings → Keybinds** (`dms ipc call settings focusOrToggle`) and save, but the file must already exist for niri to parse the include. To populate it from scratch, fetch the canonical default binds:
>
> ```bash
> curl -sL https://raw.githubusercontent.com/AvengeMedia/DankMaterialShell/master/core/internal/config/embedded/niri-binds.kdl \
>   | sed 's/{{TERMINAL_COMMAND}}/ghostty/; s/"processlist" "focusOrToggle"/"mux" "toggle"/' \
>   > ~/.config/niri/dms/binds.kdl
> ```
>
> Then verify with `dms keybinds show niri` — expect ~140 binds with `source: dms-default`, and your custom binds as `source: config`. `niri validate` must pass.

### Startup & Environment

Autostart DMS from niri (replace the `spawn-at-startup "noctalia"` line if you removed it):

```kdl
spawn-at-startup "dms" "run"
```

Niri environment block for DMS (Qt/Electron theming):

```kdl
environment {
    XDG_CURRENT_DESKTOP "niri"
    QT_QPA_PLATFORM "wayland"
    ELECTRON_OZONE_PLATFORM_HINT "auto"
    // DMS official docs use gtk3 (NOT kde) — DMS themes Qt apps itself via matugen
    QT_QPA_PLATFORMTHEME "gtk3"
    QT_QPA_PLATFORMTHEME_QT6 "gtk3"
}
```

> **DMS path: do NOT inherit the KDE-theming env vars** (`QT_QPA_PLATFORMTHEME=kde`, `QT_STYLE_OVERRIDE=breeze`) from the [KDE Integration](#kde-integration) / [Complete Environment Variables](#complete-environment-variables) sections. Those are the Noctalia path's KDE-app behavior; DMS uses `gtk3` so its own Qt UI + GTK apps follow the matugen Material theme. To theme standalone KDE apps (Dolphin, Kate, Gwenview, Ark, Spectacle) on the DMS path, use **`qt6ct-kde`** (AUR) with `QT_QPA_PLATFORMTHEME=qt6ct` — see [Qt/KDE App Theming](#qtkde-app-theming) below. (The `kde` platformtheme plugin ships with `plasma-integration`, which drags in krunner/libplasma/kscreenlocker — avoid on a Niri+DMS desktop.)

### Wallpaper & Layer Rules

DMS draws its wallpaper on the niri backdrop layer. Tell niri to place it on the overview:

```kdl
layout {
    gaps 5
    background-color "transparent"
}

layer-rule {
    match namespace="^quickshell$"
    place-within-backdrop true
}
```

If "Blur Layer" is enabled, also place the blurred wallpaper on the overview:

```kdl
layer-rule {
    match namespace="dms:blurwallpaper"
    place-within-backdrop true
}
```

### Keybinds

DMS's default keybinds live in `dms/binds.kdl` (populated above — ~140 binds: window management, workspace switching, media keys, screenshots, launchers, etc.). You do NOT need to re-bind them in `config.kdl` — that only creates duplicates. Use `dms keybinds show niri` to inspect, or **Settings → Keybinds** to edit visually.

Only add binds in `config.kdl` that DMS defaults do not provide (e.g. terminal alternative, file manager, browser — verified working pattern on pop_arch):

```kdl
binds {
    Mod+Return { spawn "ghostty"; }
    Mod+E      { spawn "dolphin"; }
    Mod+B      { spawn "firefox"; }
    Mod+Shift+Q { spawn "dms" "ipc" "call" "sessions" "open"; }
    Mod+Shift+S { spawn "spectacle" "-r"; }
    Mod+Alt+N  { spawn "dms" "ipc" "call" "night" "toggle"; }  // DMS defaults lack night toggle
}
```

DMS default keybinds you get for free (not exhaustive — verify with `dms keybinds show niri`):

| Key | Action |
|-----|--------|
| <kbd>Mod</kbd> + <kbd>T</kbd> | Terminal (`ghostty` — `{{TERMINAL_COMMAND}}` substitution) |
| <kbd>Mod</kbd> + <kbd>Space</kbd> | Spotlight launcher |
| <kbd>Alt</kbd> + <kbd>Space</kbd> | Spotlight bar |
| <kbd>Mod</kbd> + <kbd>V</kbd> | Clipboard manager |
| <kbd>Mod</kbd> + <kbd>M</kbd> | Task manager (`mux toggle`) |
| <kbd>Super</kbd> + <kbd>X</kbd> | Power menu |
| <kbd>Mod</kbd> + <kbd>Comma</kbd> | Settings |
| <kbd>Mod</kbd> + <kbd>Y</kbd> | Browse wallpapers (`dash toggle wallpaper`) |
| <kbd>Mod</kbd> + <kbd>N</kbd> | Notification center |
| <kbd>Mod</kbd> + <kbd>Shift</kbd> + <kbd>N</kbd> | Notepad |
| <kbd>Mod</kbd> + <kbd>Alt</kbd> + <kbd>L</kbd> | Lock screen |
| <kbd>Mod</kbd> + <kbd>O</kbd> / <kbd>Mod</kbd> + <kbd>Tab</kbd> | Toggle overview |
| <kbd>Mod</kbd> + <kbd>Shift</kbd> + <kbd>/</kbd> | Hotkey overlay (cheatsheet!) |
| <kbd>Mod</kbd> + <kbd>Q</kbd> | Close window |
| <kbd>Mod</kbd> + <kbd>F</kbd> | Maximize column |
| <kbd>Mod</kbd> + <kbd>H</kbd>/<kbd>J</kbd>/<kbd>K</kbd>/<kbd>L</kbd> | Focus navigation |
| <kbd>Mod</kbd> + <kbd>Shift</kbd> + <kbd>H</kbd>/<kbd>J</kbd>/<kbd>K</kbd>/<kbd>L</kbd> | Move window/column |
| <kbd>Mod</kbd> + <kbd>1</kbd>…<kbd>9</kbd> | Focus workspace |
| <kbd>Mod</kbd> + <kbd>Shift</kbd> + <kbd>1</kbd>…<kbd>9</kbd> | Move to workspace |
| <kbd>Mod</kbd> + <kbd>R</kbd> | Cycle preset column width |
| <kbd>Print</kbd> / <kbd>Ctrl</kbd>+<kbd>Print</kbd> / <kbd>Alt</kbd>+<kbd>Print</kbd> | Screenshot / output / window |
| <kbd>XF86AudioRaiseVolume</kbd>/<kbd>Lower</kbd>/<kbd>Mute</kbd> | Volume control |
| <kbd>XF86AudioPlay</kbd>/<kbd>Next</kbd>/<kbd>Prev</kbd> | MPRIS media control |
| <kbd>XF86MonBrightnessUp</kbd>/<kbd>Down</kbd> | Brightness |
| <kbd>Mod</kbd> + <kbd>Shift</kbd> + <kbd>E</kbd> | Quit niri |
| <kbd>Mod</kbd> + <kbd>Shift</kbd> + <kbd>P</kbd> | Power off monitors |

> **Verify with `dms ipc list`** — function names differ between DMS releases; this table comes from the official 1.5 docs + live `dms ipc list` on v1.5.3.

### Lock / Idle

DMS bundles lock + idle. Bind a lock key:

```kdl
binds {
    Mod+Alt+L { spawn "dms" "ipc" "call" "lock" "lock"; }
}
```

> **Confirm the lock target/function name on the live install** (`dms ipc list`) — lock IPC naming changed between DMS versions.

### Night Mode (built-in)

DMS has a built-in night mode — gamma/color-temperature control with suncalc-based scheduling (verified from DMS source: `night` IPC target with `toggle`/`temperature`/`schedule` functions). No `wlsunset` needed on this path:

```kdl
binds {
    Mod+Alt+N { spawn "dms" "ipc" "call" "night" "toggle"; }
}
```

```bash
dms ipc call night temperature 3500      # set color temperature (Kelvin)
dms ipc call night automation location   # auto schedule via suncalc (or "time")
dms ipc call night schedule 20:00 06:00  # manual time-based schedule
```

### Qt/KDE App Theming

DMS themes its own Qt UI + GTK apps via matugen with the `gtk3` platform theme. Standalone KDE apps (Dolphin, Kate, Gwenview, Ark, Spectacle) use the Qt fallback theme unless you give them a platform theme that reads KDE color schemes — verified path on pop_arch (2026-08-19):

**Why not `QT_QPA_PLATFORMTHEME=kde`:** the `kde` platformtheme plugin ships with `plasma-integration`, which pulls krunner/libplasma/kscreenlocker — heavy on a Niri+DMS desktop. **`qt6ct-kde`** (AUR) is a patched `qt6ct` that understands KDE `.colors` files — exactly what matugen generates.

```bash
# 🔒 AUR — review PKGBUILD before installing
yay -S --noconfirm qt6ct-kde
```

**1. Point qt6ct at the matugen-generated KDE scheme.** DMS already generates one (`matugenTemplateKcolorscheme: true` in Settings → `~/.config/DankMaterialShell/settings.json` → output `~/.local/share/color-schemes/DankMatugen{.colors,Light.colors,Dark.colors}`). Pick the variant matching your DMS theme mode (dark → `DankMatugenDark.colors`):

```ini
# ~/.config/qt6ct/qt6ct.conf
[Appearance]
custom_palette=true
color_scheme_path=/home/<user>/.local/share/color-schemes/DankMatugenDark.colors
```

(DMS also writes `~/.config/qt6ct/colors/matugen.conf`, but with `qt6ct-kde` point at the KDE `.colors` path above.)

**2. Set the platform theme to `qt6ct`** — the *package* is `qt6ct-kde`, but the *plugin/env value* is `qt6ct`. Set it in BOTH `~/.config/environment.d/10-kde-on-niri.conf` and the niri `environment {}` block:

```kdl
QT_QPA_PLATFORMTHEME "qt6ct"
QT_QPA_PLATFORMTHEME_QT6 "qt6ct"
```

**3. Enable DMS Qt theming.** Settings → toggle **Qt Theming** (`qtThemingEnabled: true` in `~/.config/DankMaterialShell/settings.json`). DMS's `scripts/qt.sh` writes the qt6ct config when it detects `/usr/bin/qt6ct` on PATH.

**Verify:** relaunch Dolphin — palette should match the active matugen scheme (cyan accent on flexoki dark). Note the `kdeglobals` `ColorScheme` is cosmetic-only here; `qt6ct-kde` reads the `.colors` file directly.

### Troubleshooting (DMS)

- **Shell won't start** — run `dms run` in a terminal and read the error. DMS needs `quickshell` (pulled by `dms-shell`); the tagged `quickshell` release works (unlike Caelestia which needs `quickshell-git`).
- **Missing fonts / icons** — DMS needs a Material Symbols icon font plus a Nerd Font; both are in the step 3b install (`ttf-material-symbols-variable` + `ttf-jetbrains-mono-nerd`). If icons still render as boxes after install, run `fc-cache -f`.
- **IPC returns "not running"** — `dms ipc` only works while the shell is up; ensure `spawn-at-startup "dms" "run"` fired (check with `pgrep -x dms`).
- **Customizing** — DMS keeps config under `~/.config/DankMaterialShell/`; the settings UI is `dms ipc call settings focusOrToggle`.
- **Default keybinds missing (only custom binds work)** — `dms/binds.kdl` was created empty (e.g. `touch`ed) so DMS defaults never loaded. Populate it from the canonical source (see [Niri Config Integration](#niri-config-integration)) and verify with `dms keybinds show niri` (expect ~140 binds, `source: dms-default`).
- **`dms greeter install` fails over SSH with "a terminal is required"** — the installer's internal sudo needs a TTY. Replicate its steps manually with `SUDO_ASKPASS` (see the [greeter choice](#login-manager-greeter-choice) section): `usermod -aG greeter <user>`, write `/etc/greetd/config.toml` (`command = "dms-greeter --command niri"`, `user = "greeter"`), `systemctl enable greetd`, `systemctl disable sddm`, then `dms greeter sync` (as user, NOT root) to initialize `/var/cache/dms-greeter` + theme symlinks.
- **faillock lock loop — sudo fails even with the correct password, greeter login bounces** (2026-08-19, verified on pop_arch): `sudo -A`/askpass is broken over SSH — journal shows `pam_unix(sudo:auth): conversation failed`; the askpass password never reaches `pam_unix`, so sudo fails 3× regardless of correctness. A background `yay --sudoloop --sudoflags "-A"` compounds this by failing repeatedly → 3 failures → `pam_faillock` locks the account → the greeter login also fails (system-auth is shared, `unlock_time=600`) → more failures extend the lock. **Fix:** never use `-A` on this machine — `echo "<password>" | sudo -S <cmd>`; never `yay --sudoloop -A` (build with plain `yay`, install via `sudo -S pacman -U`). If locked, reset via root `su` (pam.d/su has no faillock chain): `su -c "faillock --reset --user <user>"`, or delete the tally: `su -c "rm -f /run/faillock/<user>"`.

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
sudo pacman -Rns --noconfirm xwayland-satellite

# Verify your display manager is still the one you chose (greetd or sddm)
systemctl status greetd sddm 2>/dev/null | grep -E "●|Active"
```

**If you used the DMS (secondary) path instead of Noctalia:**

```bash
# Remove DMS shell (dms-shell-niri pulls dms-shell + dgop + quickshell)
sudo pacman -Rns --noconfirm dms-shell-niri dms-shell dgop

# If you used DankGreeter (Option A), restore the previous display manager first
dms greeter uninstall   # restores previous DM (e.g. sddm or plasmalogin)

# Remove DMS config + niri dms fragments
rm -rf ~/.config/DankMaterialShell
rm -rf ~/.config/niri/dms
rm -rf ~/.local/state/dms
rm -rf ~/.cache/dms
```

---

*Noctalia v5 is beta — expect occasional updates. Keep packages up to date: `yay -Syu noctalia-git`.*
*Docs: https://docs.noctalia.dev/v5/* 🏔️🪽
*Arch Wiki: https://wiki.archlinux.org/title/Niri*

---

## Resources

### Official Documentation

| Resource | URL |
|----------|-----|
| Niri Wiki | <https://niri-wm.github.io/niri/> |
| Niri GitHub | <https://github.com/niri-wm/niri> |
| Noctalia v5 Docs | <https://docs.noctalia.dev/v5/> |
| Noctalia v4 Docs | <https://docs.noctalia.dev/v4/> |
| Noctalia GitHub | <https://github.com/noctalia-dev/noctalia> |
| Noctalia Discord | <https://discord.noctalia.dev> |
| DMS Docs | <https://danklinux.com/docs/> |
| DMS GitHub | <https://github.com/AvengeMedia/DankMaterialShell> |

### Arch Wiki

| Page | URL |
|------|-----|
| Niri | <https://wiki.archlinux.org/title/Niri> |
| Default Applications (xdg-mime) | <https://wiki.archlinux.org/title/Default_applications> |
| Dolphin | <https://wiki.archlinux.org/title/Dolphin> |
| KDE | <https://wiki.archlinux.org/title/KDE> |

### KDE on Non-Plasma Compositors

| Resource | URL |
|----------|-----|
| **Dolphin & KDE File Picker on Niri** (linhusp) | <https://gist.github.com/linhusp/05f8f7e0af3fa0fbb944dec17a75aa78> |
| Fixing empty "Open With" in Dolphin (Hyprland → applies to Niri) | <https://www.lorenzobettini.it/2024/05/fixing-the-empty-open-with-in-dolphin-in-hyprland/> |
| CachyOS Niri + Dolphin Tweak Notes | <https://discuss.cachyos.org/t/cachyos-niri-with-dolphin-tweak-notes/32700> |
| NixOS: Dolphin MIME Associations | <https://discourse.nixos.org/t/dolphin-does-not-have-mime-associations/48985> |
| Niri Important Software (portals, keyring, etc.) | <https://github.com/niri-wm/niri/wiki/Important-Software> |

### Community Tools

| Tool | URL | Purpose |
|------|-----|---------|
| NiriMod | <https://github.com/srinivasr/nirimod> | Visual config editor for Niri |
| waycorner | <https://github.com/AndreasBackx/waycorner> | Hot corners for Wayland |
| niri_tweaks | <https://github.com/heyoeyo/niri_tweaks> | Layout scripts for advanced window arrangements |
| awesome-niri | <https://github.com/niri-wm/awesome-niri> | Curated list of Niri-related projects |

### Community

| Resource | URL |
|----------|-----|
| r/niri (Reddit) | <https://www.reddit.com/r/niri/> |
| Noctalia Discord | <https://discord.noctalia.dev> |

### Reading Material

- [Niri, btw — Setup Guide](https://www.tonybtw.com/tutorial/niri/)
- [Niri: Scrollable Tiling and the Setup Around It](https://timothyjohnsonsci.com/writing/2026-06-12-niri-setup/)
- [Custom is the new Desktop (Dunkelstern)](https://dunkelstern.de/articles/2025-01-24/index.html)
- [CachyOS Niri Keybinds & FAQ](https://wiki.cachyos.org/configuration/desktop_environments/niri/)
- [KaOS Switches to Niri with Noctalia](https://medium.com/@emilyharbord2/kaos-switches-focus-to-niri-with-noctalia-on-the-latest-iso-39d61983ad19)
