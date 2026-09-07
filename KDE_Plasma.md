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
