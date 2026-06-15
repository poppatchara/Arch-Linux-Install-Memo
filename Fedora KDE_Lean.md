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
    - [Remove PIM & System Info](#31-remove-pim--system-info)
    - [What @kde-desktop Includes](#what-kde-desktop-includes)
    - [Disable Unneeded Plasma Services](#32-disable-unneeded-plasma-services)
8. [Third-Party Repositories](#third-party-repositories)
    - [RPM Fusion](#rpm-fusion)
    - [Fedora Workstation Third-Party Repos](#fedora-workstation-third-party-repos)
    - [Terra (HP Anyware / PCoIP)](#terra-hp-anyware--teradici-pcoip-client)
9. [GPU Driver (Automatic Detection)](#gpu-driver-automatic-detection)
    - [NVIDIA Driver](#nvidia-driver)
    - [VA-API (Hardware Video Decode)](#va-api-hardware-video-decode)
    - [Verify](#verify-after-reboot-nvidia-only)
10. [Post-Install Extras](#post-install-extras)
    - [Flatpak Apps](#flatpak-apps)
    - [pyenv](#pyenv)
    - [Fonts](#fonts)
    - [Additional Apps](#44-additional-apps)
    - [Theme](#theme)
    - [SPDIF audio dropout / sleep](#spdif-audio-dropout--sleep)
    - [Clear caches](#clear-caches)
11. [Snapper (optional)](#snapper-optional)
12. [Credits & Thanks](#credits--thanks)

## Updates

### 2026-06-16

- **Restructured:** Third-party repos (`RPM Fusion`, `fedora-workstation-repositories`, Terra/PCoIP) now have their own dedicated section — separated from GPU driver.
- **VA-API:** Added hardware video decode section for Intel, AMD, and NVIDIA.
- **Removal philosophy:** Only KDE PIM (akregator/kmail/korganizer/kontact) and system info (khelpcenter/kinfocenter) removed. Everything else stays including plasma-discover, flatpak, kate, okular, gwenview, spectacle, etc.
- **Everything ISO path**: clarified software selection — Fedora 44 Anaconda renamed "Minimal Install" to **"Fedora Custom Operating System"**. Select Custom OS + Standard + KDE.
- **Btrfs subvolumes**: documented exact layout (`@`, `@home`, `@var_log`, `@var_cache`). Removed `@root` (Fedora won't allow it — lives inside `@`). Removed `@var` and `@srv` (unnecessary for desktop).
- **`@kde-desktop` group contents**: documented what the DNF group actually includes (from Fedora 44 comps). PIM apps (kontact/kmail/akregator) NOT in group — only akonadi-server backend. Everything ISO users can skip the removal section entirely (none of the 6 removed packages are in the group).
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
  fastfetch \
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

🖥️ Fedora KDE Spin ships with a lot of apps by default. This section removes only the heavy PIM suite and system info tools — everything else stays.

> **Philosophy:** Keep what you use. kate, okular, gwenview, spectacle, ark, filelight, kcalc, krfb, krdp, plasma-discover, bluedevil, plasma-firewall, flatpak, and print-manager are kept. Only KDE PIM (email/calendar/contacts) and info docs are removed.

### 3.1 Remove PIM & System Info

```bash
# Remove only the heavy PIM suite and info tools
# Everything else stays — kate, okular, gwenview, spectacle, ark, etc.

sudo dnf5 remove -y \
  akregator \
  kmail \
  korganizer \
  kontact \
  khelpcenter \
  kinfocenter
```

> **Note:** `akregator`, `kmail`, `korganizer`, `kontact` are KDE PIM apps (email, calendar, RSS, contacts) — heavy backends (akonadi + MySQL), no desktop impact when removed. `khelpcenter` and `kinfocenter` are system info/documentation viewers.

> **Everything ISO users:** If you used Custom OS + KDE, NONE of these 6 packages are installed — the entire section is a no-op.

### What @kde-desktop Includes

The `@kde-desktop` group (from Fedora 44 comps) contains 92 packages — **not** the full KDE Spin bloat. PIM apps are absent from the group. Here's what comes with it vs what doesn't:

| Cat | In `@kde-desktop`? | Packages |
|-----|-------------------|----------|
| **PIM apps** | ❌ NOT included | kontact, kmail, akregator, korganizer |
| **Info tools** | ✅ Included | khelpcenter, kinfocenter |
| **Editors** | ❌ NOT included | kate, okular, gwenview |
| **File tools** | ✅ Included | ark, filelight, spectacle |
| **Remote desktop** | ✅ Included | krfb, krdp |
| **Discover** | ✅ Included | plasma-discover, plasma-discover-notifier |
| **Bluetooth** | ✅ Included | bluedevil |
| **Print** | ✅ Included | plasma-print-manager |
| **Core** | ✅ Included | dolphin, konsole, kde-connect, plasma-nm, plasma-pa, kscreen, kscreenlocker, plasma-breeze, kwin, kcalc, plasma-firewall |

### 3.2 Disable Unneeded Plasma Services

```bash
# Disable KDE Wallet if you don't use it
mkdir -p ~/.config
cat <<'EOF' >> ~/.config/kwalletrc
[Wallet]
Enabled=false
EOF
```

## Third-Party Repositories

📦 Fedora ships with only free software in its default repos. Enable these third-party repos for drivers, codecs, and extra software.

### RPM Fusion

RPM Fusion provides software that Fedora doesn't ship — most importantly **NVIDIA drivers** and multimedia codecs.

```bash
# Enable both free and nonfree
sudo dnf5 install -y \
  https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm \
  https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm
```

> **Why nonfree?** Needed for `akmod-nvidia` (NVIDIA proprietary driver) and optional codecs. Skip `nonfree` if you have Intel/AMD iGPU only.

### Fedora Workstation Third-Party Repos

Fedora pre-configures optional repos for Google Chrome, PyCharm, and other proprietary software.

```bash
# Enable the pre-configured third-party repos
sudo dnf5 config-manager --set-enabled fedora-workstation-repositories
# Optional: enable Cisco OpenH264 codec repo
sudo dnf5 config-manager --set-enabled fedora-cisco-openh264
```

### Terra (HP Anyware / Teradici PCoIP Client)

HP Anyware PCoIP client is distributed as an Ubuntu `.deb` — no Fedora RPM repo exists. Use the dedicated install script instead:

```bash
git clone https://github.com/poppatchara/fedora-pcoip-client.git
cd fedora-pcoip-client
sudo ./install.sh
```

> No extra repository needed — the script bundles the `.deb` repackaged for Fedora.

## GPU Driver (Automatic Detection)

🟩 Auto-detects your GPU and installs the appropriate driver. **Requires RPM Fusion** (enabled above) for NVIDIA.

- **NVIDIA dGPU** → Installs `akmod-nvidia` from RPM Fusion
- **Intel/AMD iGPU** → No extra driver (kernel drivers included)

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

### NVIDIA Driver

```bash
if [ "$gpu_vendor" = "nvidia" ]; then
  # Install NVIDIA driver (akmod builds kernel module automatically)
  sudo dnf5 install -y \
    akmod-nvidia \
    xorg-x11-drv-nvidia-cuda \
    libva-utils \
    vdpauinfo

  # Wait for akmod to build the kernel module (~1-3 min)
  echo "Waiting for NVIDIA kernel module to build..."
  sudo akmods --force
  sudo dracut --force

  echo "NVIDIA driver installed. Reboot to complete."
fi
```

### VA-API (Hardware Video Decode)

```bash
# Intel
if [ "$gpu_vendor" = "intel" ]; then
  sudo dnf5 install -y intel-media-driver libva-intel-driver
fi

# AMD
if [ "$gpu_vendor" = "amd" ]; then
  sudo dnf5 install -y libva-mesa-driver
fi

# NVIDIA (needs RPM Fusion nonfree)
if [ "$gpu_vendor" = "nvidia" ]; then
  sudo dnf5 install -y libva-nvidia-driver
fi
```

### Verify (after reboot, NVIDIA only)

```bash
if [ "$gpu_vendor" = "nvidia" ]; then
  lsmod | grep nvidia
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
# Add Flathub for your user (no sudo needed)
flatpak remote-add --user --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo

# Pick what you need
flatpak install --user -y flathub \
  io.github.jonmagon.kdiskmark \
  io.missioncenter.MissionCenter \
  io.github.flattool.Warehouse \
  org.localsend.localsend_app \
  com.discordapp.Discord \
  com.vysp3r.ProtonPlus
```

> **Why `--user`?** System-wide flatpak install (`sudo flatpak install`) triggers a `revokefs-fuse` socket warning. User-scoped installs avoid this entirely and don't require root — perfect for single-user desktops.

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

### 4.4 Additional Apps

📦 Daily-driver apps from the Arch Zen guide, adapted for Fedora. RPM Fusion must be enabled.

```bash
# Media & creative
sudo dnf5 install -y vlc ffmpeg obs-studio

# Browsers
sudo dnf5 install -y firefox chromium
# Brave: flatpak install --user -y flathub com.brave.Browser

# Messaging
flatpak install --user -y flathub org.telegram.desktop

# Editor (VSCode — from Microsoft repo)
sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc
sudo sh -c 'echo -e "[code]\nname=Visual Studio Code\nbaseurl=https://packages.microsoft.com/yumrepos/vscode\ngpgcheck=1\ngpgkey=https://packages.microsoft.com/keys/microsoft.asc" > /etc/yum.repos.d/vscode.repo'
sudo dnf5 install -y code
```

> **vim, neovim, curl, wget, git, htop** — already installed in [1.3 Install Base Tools](#13-install-base-tools).

> **localsend** — already in [4.2 Flatpak Apps](#42-flatpak-apps).

> **opencode** — Hermes Agent CLI tool. Install via the Hermes setup workflow (`hermes setup`). Not a system package.

### 4.5 Theme

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

### 4.6 SPDIF audio dropout / sleep

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

### 4.7 Clear caches

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
