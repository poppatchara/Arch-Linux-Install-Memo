# KDE Plasma 🖥️

> Companion guide for the main installation guide's **KDE Plasma** path (§7).
> Pick ONE desktop path — KDE, Niri + Noctalia, or Hyprland + Celestia. Do not mix.

## Overview

Plasma 6.6+ ships `plasma-login-manager` as its native login screen (replaces SDDM). All-in-one desktop: compositor (KWin), shell, apps, integrated — install and stop.

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

## PAM config — MANDATORY

`plasma-login-manager` does not ship a default PAM file. Without this, the login screen cannot authenticate anyone:

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

> `include system-login` pulls in `pam_unix.so` — the actual password checker. Without this file, PLM has no authentication chain and login always fails with "Authentication for user '' failed".

## Desktop Apps

```bash
pacman -S --noconfirm --needed \
  dolphin dolphin-plugins konsole kate okular gwenview spectacle ark \
  gparted kio-extras ffmpegthumbs kdegraphics-thumbnailers \
  filelight kcalc btop fastfetch
```

> - `kio-extras` — SMB/SFTP/FTP support inside Dolphin's address bar
> - `ffmpegthumbs` — video thumbnails in Dolphin
> - `kdegraphics-thumbnailers` — PDF/PS/RAW photo thumbnails

## KWallet (optional)

Stores passwords for KDE apps, VS Code, Git credential helpers, and browser password sync:

```bash
pacman -S --noconfirm --needed kwallet kwalletmanager kwallet-pam
```

> Auto-unlock requires: wallet password = login password, blowfish encryption, wallet name = `kdewallet`.

## Theming (optional)

See [KDE_Theming.md](KDE_Theming.md) — Kvantum + Vinyl, WhiteSur/Orchis/Colloid themes, and the stutter note (Qogir/Darkly/Vinyl are safe; WhiteSur/Orchis/Colloid caused window-resize stutter on some hardware).
