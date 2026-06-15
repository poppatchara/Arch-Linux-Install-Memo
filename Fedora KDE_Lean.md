# Fedora 44 KDE — Lean Install Guide

Personal notes for building a lightweight Fedora 44 KDE desktop: minimal packages, Btrfs, Plasma Login Manager, and the essentials only.

*Not the Fedora Way™. Just the way I like.*

## Contents

1. [Updates](#updates)
2. [Assumptions](#assumptions)
3. [ISO Choice](#iso-choice)
4. [Installation](#installation)
    - [Btrfs Subvolume Layout](#btrfs-subvolume-layout)
    - [Installer Steps](#installer-steps)
5. [Post-Install Base Setup](#post-install-base-setup)
6. [Services & QoL](#services--qol)
7. [KDE — Make It Lighter](#kde--make-it-lighter)
    - [Remove Bloat](#31-remove-bloat)
    - [What @kde-desktop Includes](#what-kde-desktop-includes)
    - [Keep the Essentials](#32-keep-the-essentials)
    - [Disable Unneeded Plasma Services](#33-disable-unneeded-plasma-services)
8. [GPU Driver (Automatic Detection)](#gpu-driver-automatic-detection)
9. [Post-Install Extras](#post-install-extras)
    - [RPM Fusion + Third-Party Repos](#rpm-fusion--third-party-repos)
    - [Flatpak Apps](#flatpak-apps)
    - [pyenv](#pyenv)
    - [Fonts](#fonts)
    - [Theme](#theme)
    - [SPDIF audio dropout / sleep](#spdif-audio-dropout--sleep)
    - [Clear caches](#clear-caches)
10. [Snapper (optional)](#snapper-optional)
11. [Credits & Thanks](#credits--thanks)

## Updates

### 2026-06-16

- **Everything ISO path**: clarified software selection — Fedora 44 Anaconda renamed "Minimal Install" to **"Fedora Custom Operating System"**. Select Custom OS + Standard + KDE.
- **Btrfs subvolumes**: documented exact layout (`@`, `@home`, `@var_log`, `@var_cache`). Removed `@root` (Fedora won't allow it — lives inside `@`). Removed `@var` and `@srv` (unnecessary for desktop).
- **`@kde-desktop` group contents**: documented what the DNF group actually includes (from Fedora 44 comps). PIM apps (kontact/kmail/akregator) NOT in group — only akonadi-server backend. Good news: half the removal list is unnecessary when using Everything ISO.
- `/boot` on Btrfs: Fedora 44 Cloud defaults to `/boot` as Btrfs subvolume; KDE/Workstation still use separate ext4 `/boot` by default. Custom partitioning can merge `/boot` into `@`.

### 2026-06-07

- Initial release: Fedora 44 KDE lean install guide.
- Same philosophy as Arch variants: auto-detect GPU, minimal bloat, Btrfs subvolumes.

## Assumptions

🧩 Fedora uses the Anaconda GUI installer. This guide focuses on post-install setup — the things you do after the first boot to make the system lean and functional.

- Fedora 44 KDE Plasma Spin (or Everything ISO with `@kde-desktop` group).
- Secure Boot disabled, UEFI mode enabled.
- Btrfs is Fedora's default filesystem — no extra setup needed.
- Timezone `Asia/Bangkok`; hostname defaults to `fedora`.
- Wired Ethernet or reliable Wi-Fi available.

## ISO Choice

### Option A: KDE Plasma Spin (Recommended)

Download: <https://fedoraproject.org/en/spins/kde>

- Smallest ISO that gives you KDE out of the box.
- During install: select "Custom" partitioning if you want custom subvolumes.

### Option B: Everything / Netinstall ISO

Download: <https://fedoraproject.org/en/server/download>

- Minimal base install, then pick exactly what you install.
- In Anaconda → Software Selection:
  - **Base Environment:** ☑ **Fedora Custom Operating System** (this is the minimal base — Anaconda renamed it from "Minimal Install" in Fedora 44)
  - **Additional Software:** ☑ Standard (`@standard`), ☑ KDE (Plasma Workspaces) (`@kde-desktop`)
  - Uncheck everything else (Office, Print Server, Web Server, etc.)
- **Why this over KDE Spin:** `@kde-desktop` group does NOT include KDE PIM apps (kontact/kmail/akregator/korganizer) — they won't be installed at all. Saves you from removing them later. See [What @kde-desktop includes](#what-kde-desktop-includes).

## Installation

### Btrfs Subvolume Layout

📦 Fedora defaults to Btrfs with `@` and `@home` subvolumes. For Snapper-compatible snapshots, use this custom layout:

| Subvolume | Mount Point | Snapshot? | Notes |
|-----------|------------|-----------|-------|
| `@` | `/` | ✅ YES | Root + kernel via `/boot` merged |
| `@home` | `/home` | ❌ NO | User data |
| `@var_log` | `/var/log` | ❌ NO | Logs |
| `@var_cache` | `/var/cache` | ❌ NO | DNF5 cache |

**Why not `@var`?** Fedora stores Flatpak, containers, and other large data under `/var/lib` — snapshotting all of `/var` is too heavy. Split out only what matters.

**Why not `@root`?** Anaconda won't allow a separate `/root` mount. `/root` lives inside `@` — it's tiny anyway (~0 bytes on a fresh install).

**`/boot` on Btrfs?** Fedora 44 Cloud defaults to `/boot` on Btrfs. KDE/Workstation still use separate ext4 `/boot` by default. In Custom partitioning you can skip the `/boot` partition entirely and let `/boot` live inside `@` — this way Snapper snapshots include the kernel, so rollback keeps everything in sync. GRUB2 2.12+ (Fedora 44) supports Btrfs zstd compression.

### Installer Steps

1. Boot ISO → Anaconda GUI installer.
2. **Keyboard**: Set to your layout.
3. **Installation Destination**: Select disk. Fedora defaults to Btrfs with `@` and `@home` subvolumes — this is fine for most users.
4. **Network & Hostname**: Set hostname to `fedora` (or your choice), enable Ethernet/Wi-Fi.
5. **Root / User**: Create your user, add to `wheel` group, set root password.
6. **Begin Installation** → Wait → Reboot.

## Post-Install Base Setup

🧰 Goal: configure DNF, mirrors, and basic tools before adding packages.

### 1.1 Speed up DNF5

Fedora 44 ships with `dnf5` by default. Speed it up:

```bash
# DNF5 doesn't need max_parallel_downloads tuning like dnf4,
# but fastestmirror helps on slower connections.

sudo dnf5 config-manager --setopt fastestmirror=True
```

### 1.2 Update Everything First

```bash
sudo dnf5 upgrade --refresh
```

### 1.3 Install Base Tools

```bash
sudo dnf5 install -y \
  git vim neovim curl wget htop btop \
  tmux zsh zsh-autosuggestions \
  rsync bat unzip p7zip \
  NetworkManager-tui openssh-server \
  pipewire pipewire-pulseaudio wireplumber \
  xdg-user-dirs
```

### 1.4 Enable Essential Services

```bash
sudo systemctl enable --now sshd
sudo systemctl enable --now fstrim.timer
```

### 1.5 SSH Setup (Optional)

```bash
# Generate key on client, then:
# ssh-copy-id user@fedora

# Harden: disable password auth after key works
sudo sed -i 's/^#\?PasswordAuthentication .*/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo systemctl restart sshd
```

## Services & QoL

### 2.1 Optional Packages

Install only what you need:

```bash
sudo dnf5 install -y \
  util-linux inetutils usbutils \
  iwd \
  avahi avahi-tools \
  alsa-utils sof-firmware \
  bluez bluez-tools \
  cups \
  acpid
```

### 2.2 Enable Services

```bash
# Pick what you need
sudo systemctl enable --now bluetooth
# sudo systemctl enable --now cups
sudo systemctl enable --now avahi-daemon
```

## KDE — Make It Lighter

🖥️ Fedora KDE Spin ships with a lot of apps by default. Strip the fat.

> **Everything ISO users:** If you used Custom OS + KDE, about half the removal list below is already NOT installed — skip to the [notes](#what-kde-desktop-includes) to see what's different.

### 3.1 Remove Bloat

```bash
# Remove apps you don't need (adjust to taste)

sudo dnf5 remove -y \
  akregator \
  kmail \
  korganizer \
  kontact \
  khelpcenter \
  kinfocenter \
  kate \
  okular \
  gwenview \
  spectacle \
  ark \
  filelight \
  kcalc \
  krfb \
  krdp \
  plasma-discover \
  kde-print-manager \
  bluedevil \
  plasma-firewall \
  flatpak \
  xdg-desktop-portal-gnome
```

> **Note about flatpak:** `flatpak` is removed here (the KDE Spin includes it), but reinstalled later in [4.2 Flatpak Apps](#42-flatpak-apps) with Flathub configured. If using Everything ISO, `flatpak` may not be installed yet — skip it in the remove list if `dnf5 remove` complains.

> **Note:** This removes a lot. If you use any of these apps, keep them. The point is: you don't need a full KDE suite to run KDE Plasma.

### What @kde-desktop Includes

The `@kde-desktop` group (from Fedora 44 comps) contains 92 packages — **not** the full KDE Spin bloat. Good news: PIM apps are absent. Bad news: some removal candidates still come with the group.

| Cat | In `@kde-desktop`? | Packages |
|-----|-------------------|----------|
| **PIM apps** | ❌ NOT included | kontact, kmail, akregator, korganizer — skip these removals |
| **Editors** | ❌ NOT included | kate, okular, gwenview — skip these removals |
| **System info** | ✅ Included | khelpcenter, kinfocenter — still need to remove |
| **Remote desktop** | ✅ Included | krfb, krdp — still need to remove |
| **Discover** | ✅ Included | plasma-discover, plasma-discover-notifier — still need to remove |
| **Bluetooth** | ✅ Included | bluedevil — still need to remove |
| **Print** | ✅ Included | plasma-print-manager — still need to remove |
| **File tools** | ✅ Included | ark, filelight, spectacle — still need to remove |
| **Core** | ✅ Included | dolphin, konsole, kde-connect, plasma-nm, plasma-pa, kscreen, kscreenlocker, plasma-breeze, kwin |

**Bottom line:** Everything ISO + `@kde-desktop` starts ~30% cleaner than KDE Spin.

### 3.2 Keep the Essentials

```bash
# Reinstall only what you actually want from the list above

sudo dnf5 install -y \
  dolphin konsole \
  kde-connect \
  xdg-desktop-portal-kde
```

### 3.3 Disable Unneeded Plasma Services

```bash
# Disable Discover notifier (you removed discover anyway)
systemctl --user disable --now plasma-discover-notifier.service 2>/dev/null || true

# Disable KDE Wallet if you don't use it
mkdir -p ~/.config
cat <<'EOF' >> ~/.config/kwalletrc
[Wallet]
Enabled=false
EOF
```

## GPU Driver (Automatic Detection)

🟩 This section auto-detects your GPU and installs the appropriate driver.

- **NVIDIA dGPU** → Installs `akmod-nvidia` from RPM Fusion (proprietary)
- **Intel/AMD iGPU** → No extra driver needed (kernel drivers included)

```bash
# Detect GPU vendor

gpu_vendor=""
if lspci | grep -qi nvidia; then
  gpu_vendor="nvidia"
elif lspci | grep -qi "intel.*graphic\|intel.*display\|intel.*UHD\|intel.*Iris\|intel.*Arc"; then
  gpu_vendor="intel"
elif lspci | grep -qi "amd.*graphic\|amd.*radeon\|amd.*advanced"; then
  gpu_vendor="amd"
fi

echo "Detected GPU vendor: ${gpu_vendor:-unknown}"
```

### RPM Fusion + NVIDIA (if detected)

```bash
if [ "$gpu_vendor" = "nvidia" ]; then
  # Enable RPM Fusion (required for NVIDIA proprietary driver)
  sudo dnf5 install -y \
    https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm \
    https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm

  # Enable Fedora Linux Third Party Repo (optional, for multimedia codecs)
  sudo dnf5 config-manager --set-enabled fedora-cisco-openh264

  # Install NVIDIA driver (akmod builds for current kernel automatically)
  sudo dnf5 install -y \
    akmod-nvidia \
    xorg-x11-drv-nvidia-cuda \
    libva-utils \
    vdpauinfo

  # Wait for akmod to build the kernel module (takes 1-3 minutes)
  echo "Waiting for NVIDIA kernel module to build..."
  sudo akmods --force
  sudo dracut --force

  echo "NVIDIA driver installed. Reboot to load the nouveau blacklist and nvidia module."
fi
```

### Verify (after reboot, NVIDIA only)

```bash
if [ "$gpu_vendor" = "nvidia" ]; then
  # Check if the nvidia module is loaded
  lsmod | grep nvidia

  # Check if nvidia-smi works
  nvidia-smi
fi
```

## Post-Install Extras

### 4.1 pyenv

🐍 Goal: install `pyenv` so you can manage multiple Python versions per-user.

```bash
# Install build deps
sudo dnf5 install -y \
  gcc make cmake \
  openssl-devel zlib-devel bzip2-devel \
  readline-devel sqlite-devel xz-devel \
  tk-devel libffi-devel

# Clone pyenv
git clone https://github.com/pyenv/pyenv.git ~/.pyenv

# Shell integration
shell_name="$(basename "${SHELL:-bash}")"
conf="$HOME/.bashrc"
if [ "$shell_name" = "zsh" ]; then conf="$HOME/.zshrc"; fi

if ! grep -qF 'PYENV_ROOT' "$conf"; then
  cat <<'EOF' >> "$conf"

# pyenv
export PYENV_ROOT="$HOME/.pyenv"
export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init -)"
EOF
fi

# Install Python 3.13 (or your preferred version)
source "$conf"
pyenv install --list | grep " 3\.13"
pyenv install 3.13.2
pyenv global 3.13.2
```

### 4.2 Flatpak Apps

```bash
# Reinstall flatpak if you removed it earlier
sudo dnf5 install -y flatpak
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo

# Pick what you need
flatpak install -y flathub \
  io.github.jonmagon.kdiskmark \
  io.missioncenter.MissionCenter \
  io.github.flattool.Warehouse \
  org.localsend.localsend_app \
  com.discordapp.Discord \
  com.vysp3r.ProtonPlus
```

### 4.3 Fonts

```bash
sudo dnf5 install -y \
  google-noto-sans-fonts \
  google-noto-sans-cjk-fonts \
  google-noto-emoji-fonts \
  jetbrains-mono-fonts \
  fira-code-fonts \
  dejavu-sans-mono-fonts \
  liberation-mono-fonts \
  terminus-fonts
```

### 4.4 Theme

🎨 KDE settings path: **System Settings → Appearance**.

Fedora KDE uses Breeze by default. For custom themes:

```bash
# Install theme tools
sudo dnf5 install -y kvantum

# Popular themes (download manually from KDE Store or GitHub)
# Example: install Qogir KDE theme
# git clone https://github.com/vinceliuice/Qogir-kde.git ~/Desktop/theme/Qogir-kde
# cd ~/Desktop/theme/Qogir-kde && ./install.sh
```

> **Note:** Fedora 44 KDE uses Plasma Login Manager by default (Plasma 6.6+). Theme the login screen via System Settings → Login Screen.

### 4.5 SPDIF audio dropout / sleep

🔊 Same fix as Arch — disable codec autosuspend:

```bash
sudo tee /etc/modprobe.d/alsa-no-powersave.conf >/dev/null <<'EOF'
options snd_hda_intel power_save=0 power_save_controller=N
EOF

# WirePlumber: stop sink suspend
mkdir -p ~/.config/wireplumber/main.lua.d
cat <<'EOF' > ~/.config/wireplumber/main.lua.d/51-spdif-nosuspend.lua
alsa_monitor.rules = alsa_monitor.rules or {}

table.insert(alsa_monitor.rules, {
  matches = {
    { { "node.name", "matches", "alsa_output.*" } },
  },
  apply_properties = {
    ["session.suspend-timeout-seconds"] = 0
  }
})
EOF

systemctl --user restart wireplumber
```

### 4.6 Clear caches

```bash
sudo dnf5 clean all
```

## Snapper (Optional)

📸 Fedora uses Btrfs by default. Set up snapshots for rollback.

```bash
# Install snapper + Btrfs tools
sudo dnf5 install -y snapper btrfs-assistant

# Create snapper config for root
sudo snapper -c root create-config /

# Enable snapper timer
sudo systemctl enable --now snapper-timer.timer

# Optional: grub-btrfs for GRUB snapshot integration
# Note: grub2-snapper-plugin is not in Fedora repos; use grub-btrfs + inotify-tools instead
# sudo dnf5 install -y grub-btrfs inotify-tools
```

## Credits & Thanks

🙏 Thanks to the Fedora community and these references:

- Fedora KDE Spin: <https://fedoraproject.org/en/spins/kde>
- RPM Fusion: <https://rpmfusion.org>
- NVIDIA on Fedora: <https://rpmfusion.org/Howto/NVIDIA>
- Fedora Btrfs defaults: <https://fedoraproject.org/wiki/Changes/BtrfsByDefault>
