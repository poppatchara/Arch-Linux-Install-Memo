# KDE Plasma 🖥️

> Companion guide for the main installation guide's **KDE Plasma** path (§7).
> Pick ONE desktop path — KDE, Niri + Noctalia, or Hyprland + Celestia. Do not mix.

## Overview

Plasma 6.6+ ships `plasma-login-manager` as its native login screen (replaces SDDM). All-in-one desktop: compositor (KWin), shell, apps, integrated — install and stop. *Want the pixie (Pixel UI) login screen instead? See [Alternative login: SDDM + pixie](#alternative-login-sddm--pixie-optional).*

## Installation

```bash
# Core login + Wayland
pacman -S --noconfirm --needed \
  plasma-login-manager \
  xdg-desktop-portal xdg-desktop-portal-kde \
  qt6-wayland xorg-xwayland

# Plasma desktop (pulls plasma-workspace, kwin, systemsettings)
pacman -S --noconfirm --needed \
  plasma-desktop \
  plasma-nm plasma-pa kscreen \
  kde-gtk-config breeze-gtk

# Optional Plasma extras
pacman -S --noconfirm --needed \
  bluedevil power-profiles-daemon \
  kdeplasma-addons plasma-systemmonitor \
  plasma-browser-integration discover \
  krdp print-manager

systemctl enable plasmalogin
systemctl enable power-profiles-daemon
```

> - `plasma-nm` — NetworkManager applet (Wi-Fi selection, VPN)
> - `plasma-pa` — audio volume applet
> - `kscreen` — monitor hotplug handling
> - `kde-gtk-config breeze-gtk` — makes GTK apps (Firefox, GIMP) match KDE's theme
> - `power-profiles-daemon` — laptop power modes (balanced/powersave/performance)

## PAM config — auto-provided since Plasma 6.7

`plasma-login-manager` ≥ 6.7 ships its PAM config in the package (vendor dir):

```
/usr/lib/pam.d/plasmalogin           # standard login
/usr/lib/pam.d/plasmalogin-autologin # auto-login variant
/usr/lib/pam.d/plasmalogin-greeter   # greeter auth
```

Arch's `libpam` (≥ 1.5) reads `/usr/lib/pam.d` as a fallback when no `/etc/pam.d/<service>` exists (verified on pam 1.7.2-2 — search path includes both dirs; `polkit-1` relies on the vendor dir alone and works). So **no manual PAM file is needed** for a stock install — the shipped `plasmalogin` file includes `system-login` (the actual password checker) plus optional `pam_kwallet5.so` / `pam_gnome_keyring.so` hooks.

Only create an `/etc/pam.d/plasmalogin` override if you need something the shipped file lacks — e.g. **passwordless login for members of the `nopasswdlogin` group**, or pinning `kwalletd=/usr/bin/ksecretd` explicitly:

```bash
tee /etc/pam.d/plasmalogin <<'EOF'
#%PAM-1.0
auth       sufficient   pam_succeed_if.so user ingroup nopasswdlogin
auth       include      system-login
account    include      system-login
session    include      system-login
session    optional     pam_kwallet5.so auto_start kwalletd=/usr/bin/ksecretd
password   include      system-login
EOF
```

> `/etc/pam.d/plasmalogin` shadows the vendor file — don't create it unless you actually need the override.
> KWallet auto-unlock (optional): wallet password = login password, blowfish encryption, wallet name = `kdewallet`.

## Alternative login: SDDM + pixie (optional)

`plasma-login-manager` is the default above — native, ships its own PAM, KWallet hooks included. Its one limitation: **it does not support custom QML themes** (appearance = Plasma color scheme + wallpaper only, applied via *Apply Plasma Settings* in the login-screen KCM).

If you want a **pixie** login screen (Pixel UI / Material Design 3 — the same theme the Niri/Hyprland paths use), swap the display manager to SDDM instead:

```bash
# SDDM (official) — must be >= 0.20.0 to avoid bug #1476 (90s shutdowns).
# Arch's package is current (0.21+), so this is already satisfied.
sudo pacman -S --noconfirm --needed sddm

# pixie theme (AUR, theme author xCaptaiN09) — Material 3 / Pixel UI
# 🔒 AUR — review `yay -G pixie-sddm-git` before installing if desired.
yay -S --noconfirm --needed pixie-sddm-git
```

> **On the KDE path nothing extra is needed:** `kwin`, `layer-shell-qt`, and the Qt6 engine (`qt6-declarative`/`qt6-svg`) already come with `plasma-desktop` — the Wayland greeter and QML theme both work out of the box. (The Niri path has to install those separately; you don't.)

Point SDDM at pixie and switch to the Wayland greeter (drops the Xorg greeter — saves ~135MB, and tears down cleanly with the session):

```bash
sudo sh -c 'echo "\
[Theme]\
Current=pixie" > /etc/sddm.conf.d/theme.conf'

sudo tee /etc/sddm.conf.d/01-wayland.conf > /dev/null <<'EOF'
[General]
DisplayServer=wayland
GreeterEnvironment=QT_WAYLAND_SHELL_INTEGRATION=layer-shell,XCURSOR_THEME=phinger-cursors-dark

[Wayland]
CompositorCommand=kwin_wayland --drm --no-lockscreen --no-global-shortcuts --locale1
EOF
```

> **Config source:** [Arch Wiki — SDDM §Wayland → KDE Plasma/KWin](https://wiki.archlinux.org/title/SDDM#KDE_Plasma_/_KWin). `CompositorCommand` needs the full flag set (`--drm` = KMS/DRM rendering, `--no-global-shortcuts` drops the KWin shortcut layer, `--locale1` reads the layout from `localectl`); `--no-lockscreen` alone is not enough. `GreeterEnvironment=QT_WAYLAND_SHELL_INTEGRATION=layer-shell` makes QML themes composite as layer-shell surfaces — without it the greeter may not render. `XCURSOR_THEME` is **required**: sddm injects `GreeterEnvironment` verbatim and its packaged default leaves the cursor theme empty → the Wayland greeter shows no cursor at all.

**PAM — already shipped:** Arch's `sddm` package ships `/etc/pam.d/sddm` (system-login + `pam_kwallet5`/`pam_gnome_keyring` hooks) — no manual PAM file needed on a stock install. Verify it exists and has the KWallet hook:

```bash
ls -la /etc/pam.d/sddm
grep -i kwallet /etc/pam.d/sddm   # expect: -session optional pam_kwallet5.so ...
```

Test on a spare VT, then switch:

```bash
# Do NOT disable plasmalogin yet — test first (leave your session running)
sudo systemctl start sddm
```

Switch to TTY1 — pixie login screen with the **Plasma** session entry (SDDM lists every `*.desktop` in `/usr/share/wayland-sessions/`, so `plasma.desktop` appears automatically). If it works, switch back and enable:

```bash
sudo systemctl enable sddm --now
sudo systemctl disable plasmalogin
sudo reboot
```

**Rollback** (from a TTY, Ctrl+Alt+F3):

```bash
sudo rm /etc/sddm.conf.d/01-wayland.conf && sudo systemctl restart sddm   # back to X11 greeter
sudo pacman -S --noconfirm plasma-login-manager
sudo systemctl disable sddm
sudo systemctl enable plasmalogin
sudo reboot
```

> Theming note: pixie only works on SDDM. If you stay on `plasma-login-manager`, the login screen follows your Plasma theme + wallpaper (see [KDE_Theming.md](KDE_Theming.md)) — no QML login themes supported.

## Desktop Apps

```bash
pacman -S --noconfirm --needed \
  dolphin dolphin-plugins konsole kate okular gwenview spectacle ark unrar \
  gparted kio-extras ffmpegthumbs kdegraphics-thumbnailers \
  filelight kcalc btop fastfetch
```

> - `kio-extras` — SMB/SFTP/FTP support inside Dolphin's address bar
> - `ffmpegthumbs` — video thumbnails in Dolphin
> - `kdegraphics-thumbnailers` — PDF/PS/RAW photo thumbnails
> - `unrar` — RAR extraction backend for Ark/Dolphin (`unrar x file.rar`). Without it, `.rar` extract fails in the GUI even though 7z/zip work (bsdtar/libarchive handles only a subset of RARs).

## KWallet (optional)

Stores passwords for KDE apps, VS Code, Git credential helpers, and browser password sync:

```bash
pacman -S --noconfirm --needed kwallet kwalletmanager kwallet-pam
```

> Auto-unlock requires: wallet password = login password, blowfish encryption, wallet name = `kdewallet`.

## Theming (optional)

See [KDE_Theming.md](KDE_Theming.md) — Kvantum + Vinyl, WhiteSur/Orchis/Colloid themes, and the stutter note (Qogir/Darkly/Vinyl are safe; WhiteSur/Orchis/Colloid caused window-resize stutter on some hardware).
