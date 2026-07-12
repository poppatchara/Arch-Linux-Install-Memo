# Arch Linux + Btrfs + linux-zen + GRUB + Niri + Noctalia v5 + NVIDIA

Personal notes for rebuilding my daily Arch install: UEFI firmware, single NVMe drive, Btrfs root, Niri compositor + Noctalia v5 shell, KDE apps integration, GRUB bootloader, Snapper snapshots, and NVIDIA driver.

Not the best way or most correct way. Just the way I like.

> **Niri+Noctalia companion:** See [Niri_Noctalia_v5.md](Niri_Noctalia_v5.md) for detailed config, keybinds, and post-install tweaks.

## Contents

1. [Updates](#updates)
2. [Assumptions](#assumptions)
3. [Live ISO Prep](#live-iso-prep)
4. [Partition & Format](#partition--format)
5. [Btrfs Subvolumes & Mounts](#btrfs-subvolumes--mounts)
6. [Base Install](#base-install)
7. [Chroot Configuration](#chroot-configuration)
8. [GRUB Bootloader](#grub-bootloader)
9. [Services & QoL](#services--qol)
10. [Desktop Stack](#desktop-stack)
11. [Reboot](#reboot)
12. [Post-Install](#post-install)
13. [SSH Setup](#ssh-setup)
14. [YAY Package Manager](#yay-package-manager)
15. [Snapper](#snapper)
16. [Extra Packages](#extra-packages)
17. [GPU Driver](#gpu-driver)
18. [KDE Integration](#kde-integration)
19. [Niri + Noctalia Post-Config](#niri--noctalia-post-config)
20. [pyenv](#pyenv)
21. [Flatpak Apps](#flatpak-apps)
22. [Theme](#theme)

## Updates

### 2026-07-13

- Initial version — forked from `Arch Linux_Zen_Grub_Btrfs.md`
- Replaced KDE Plasma desktop with Niri + Noctalia v5
- Uses `greetd` + `noctalia-greeter` instead of `plasma-login-manager`
- KDE apps (Dolphin, Kate, etc.) still installed with minimal integration
- No `plasmashell`, `plasma-workspace`, `kwin` — lighter than full KDE

## Assumptions

🧩 This memo is written as a linear "do this, then that" install. If you deviate (different disk name, multiple disks, LUKS, existing Windows ESP, etc.), adjust the affected commands and double-check mounts/UUIDs before continuing.

- Arch ISO already written to USB (any recent release).
- I SSH into the live ISO from another machine (root password set with `passwd`, IP from `ip a`), purely for copy/paste convenience.
- Secure Boot disabled, UEFI mode enabled.
- Target disk: `/dev/nvme0n1` (adjust commands if yours differs).
- Wired Ethernet or reliable Wi-Fi (`iwctl`) available.
- Timezone `Asia/Bangkok`; hostname defaults to `arch`.
- EFI System Partition (ESP) already exists as the first partition; swap is the last partition to simplify potential resizing.

## Live ISO Prep

🧰 Goal: make package installs fast/repeatable in the live environment (pacman config + mirrors) before you partition/format. On the Arch ISO you are usually `root` already; `sudo` is fine to keep the same commands for later.

### 0.1 Configure pacman appearance & parallel downloads 🎛️

```bash
conf=/etc/pacman.conf

# Bump ParallelDownloads from 5→15 and enable colorized pacman output.

perl -pi -e '
  s/^(ParallelDownloads\s*=\s*)5/${1}15/;
  s/^#Color/Color/;
' "$conf"

# Uncomment `[multilib]` and its `Include = /etc/pacman.d/mirrorlist` line near the end of the file.

perl -0777 -pi -e '
s/^#\[(multilib)\]\n#(Include\s*=\s*\/etc\/pacman\.d\/mirrorlist)(\n)/[\1]\n\2\3/mg
' "$conf"
pacman -Syy

```

### 0.2 Refresh mirror list 🌐

Reorder the existing mirrorlist: move your `## <country>` section to right before `## Worldwide`.
Change `country=` to yours.

```bash
country=Thailand
mirrorfile="/etc/pacman.d/mirrorlist"
cp "$mirrorfile" "${mirrorfile}.bak"

tmpfile="$(mktemp)"
awk -v country="$country" '
  { lines[NR] = $0; n = NR }
  END {
    country_re   = "^##[[:space:]]+" country "[[:space:]]*$"
    worldwide_re = "^##[[:space:]]+Worldwide[[:space:]]*$"

    for (i = 1; i <= n; ++i) if (lines[i] ~ country_re)   { cs = i; break }
    for (i = 1; i <= n; ++i) if (lines[i] ~ worldwide_re) { ws = i; break }
    if (!cs) { print "Country section not found: ## " country > "/dev/stderr"; exit 2 }
    if (!ws) { print "Worldwide section not found: ## Worldwide" > "/dev/stderr"; exit 2 }

    ce = n + 1
    for (i = cs + 1; i <= n; ++i) {
      if (lines[i] ~ /^##[[:space:]]+/) { ce = i; break }
    }

    m = 0
    for (i = cs; i < ce; ++i) { c[++m] = lines[i] }
    while (m > 0 && c[m] ~ /^[[:space:]]*$/) { --m }

    for (i = 1; i < ws; ++i) {
      if (i < cs || i >= ce) print lines[i]
    }
    for (i = 1; i <= m; ++i) print c[i]
    print ""
    for (i = ws; i <= n; ++i) {
      if (i < cs || i >= ce) print lines[i]
    }
  }
' "$mirrorfile" > "$tmpfile" || { echo "mirrorlist reorder failed; restoring backup." >&2; cp "${mirrorfile}.bak" "$mirrorfile"; rm -f "$tmpfile"; exit 1; }

cp "$tmpfile" "$mirrorfile"
rm -f "$tmpfile"

pacman -Syy

```

## Partition & Format

💽 Create a minimal GPT layout (ESP + Btrfs root + swap) and capture stable UUIDs for later bootloader and `fstab` configuration. Before running format commands, sanity-check your target disk with `lsblk -f` so you don't wipe the wrong device.

### 1. My Partition layout

| Partition | Size | Type | Purpose |
|-----------|------|------|---------|
| `/dev/nvme0n1p1` | 2-4 GB | EFI System (type `ef00`) | Mounted at `/boot` |
| `/dev/nvme0n1p2` | About half or equal your RAM size | Linux swap | Swap |
| `/dev/nvme0n1p3` | Remainder | Linux filesystem (`8300`) | Btrfs root |

You can use `cfdisk /dev/nvme0n1` (GPT) to build or adjust the layout.

**Using cfdisk:**

1. Run `cfdisk /dev/nvme0n1`
2. Create partition 1: New → 2-4G → Type → EFI System (p1)
3. Create partition 2: New → (swap size) → Type → Linux swap (p2)
4. Create partition 3: New → (remainder) → Type → Linux filesystem (p3)
5. Write → type `yes` → Quit

### 1.1 Format EFI partition (optional, single-boot only) ⚠️

**Only format the EFI partition if this is a fresh single-boot install.** If you are dual-booting (e.g., Windows already installed), **skip this step** to preserve the existing ESP and Windows bootloader.

```bash

# WARNING: This will erase all data on the EFI partition!
# Do NOT run this if you have Windows or another OS on the same ESP.

mkfs.fat -F32 -n EFI /dev/nvme0n1p1

```

### 2. Format and capture UUIDs

Setup the Btrfs root partition, enable swap, and capture UUIDs for fstab/GRUB:

```bash

# Format the Btrfs root partition

mkfs.btrfs -f -L Arch /dev/nvme0n1p3

# Initialize swap

mkswap /dev/nvme0n1p2
swapon /dev/nvme0n1p2

# Auto-detect ESP by GPT partition type GUID

esp_part="$(lsblk -no PATH,PARTTYPE | while read -r part type; do
  case "$type" in
    c12a7328-f81f-11d2-ba4b-00a0c93ec93b) echo "$part"; break ;;
  esac
done)"
if [ -z "$esp_part" ]; then
  echo "ERROR: No EFI System Partition found" >&2
  exit 1
fi
echo "Detected ESP: $esp_part"
esp_uuid="$(blkid -s UUID -o value "$esp_part")"

# Auto-detect swap partition (any disk)
swap_part="$(blkid -t TYPE=swap -o device | head -1)"
if [ -z "$swap_part" ]; then
  echo "ERROR: No swap partition found" >&2
  exit 1
fi
echo "Detected swap partition: $swap_part"
swap_uuid="$(blkid -s UUID -o value "$swap_part")"

btrfs_root_uuid="$(blkid -s UUID -o value /dev/nvme0n1p3)"

echo "ESP UUID:      $esp_uuid"
echo "swap UUID:     $swap_uuid"
echo "Btrfs UUID:    $btrfs_root_uuid"

```

## Btrfs Subvolumes & Mounts

🧱 Goal: create a subvolume layout that plays nicely with snapshot tools (e.g., Snapper) and keeps "churny" paths isolated (logs/cache). This is a personal layout; feel free to add/remove subvolumes depending on what you plan to snapshot.

```bash

# Mount Btrfs root

mount /dev/nvme0n1p3 /mnt

# Create subvolumes

btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@var_log
btrfs subvolume create /mnt/@var_cache
btrfs subvolume create /mnt/@swap

```

> **Why no `@var`?** Fedora stores Flatpak, containers under `/var/lib` — snapshotting all of `/var` is too heavy.

Then unmount, mount subvolumes with options, and prepare `/boot`:

```bash
umount /mnt

# Mount subvolumes
mount -o compress=zstd,subvol=@ /dev/nvme0n1p3 /mnt

mkdir -p /mnt/{efi,home,var/log,var/cache,swap,.snapshots}

mount -o compress=zstd,subvol=@home /dev/nvme0n1p3 /mnt/home
mount -o compress=zstd,subvol=@var_log /dev/nvme0n1p3 /mnt/var/log
mount -o compress=zstd,subvol=@var_cache /dev/nvme0n1p3 /mnt/var/cache
mount -o subvol=@swap /dev/nvme0n1p3 /mnt/swap

# Mount ESP to /boot (Btrfs root + kernel via /boot merged)
mkdir -p /mnt/boot
mount /dev/nvme0n1p1 /mnt/boot

```

## Base Install

🧰 Goal: install the essential Arch base system onto the new Btrfs layout.

```bash

# Base system + Btrfs tools

pacstrap -K /mnt base linux-zen linux-firmware btrfs-progs efibootmgr dosfstools base-devel \
  networkmanager \
  zram-generator \
  git vim nvim curl \
  man-db man-pages \
  pipewire wireplumber pipewire-alsa pipewire-pulse pipewire-jack

genfstab -U /mnt >> /mnt/etc/fstab

```

## Chroot Configuration

⚙️ Goal: configure locale, timezone, hostname, username and password, and mkinitcpio inside the chroot environment.

### 5.0 Chroot Safety Check

```bash
# Live ISO uses overlay/squashfs; installed system uses Btrfs
if [ "$(findmnt -n -o FSTYPE /)" != "btrfs" ]; then
  echo "ERROR: Not in chroot — / is not on Btrfs." >&2
  echo "Run 'arch-chroot /mnt' first." >&2
  exit 1
fi
```

### 5.1 Enter chroot

```bash
arch-chroot /mnt
```

### 5.2 Locale, Timezone, Hostname, User

```bash
# Locale
echo "en_US.UTF-8 UTF-8" > /etc/locale.gen
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf

# Timezone
ln -sf /usr/share/zoneinfo/Asia/Bangkok /etc/localtime
hwclock --systohc

# Hostname
echo "arch" > /etc/hostname

# Root password
passwd

# Create user (with docker group pre-created for later)
groupadd -f docker
useradd -m -G wheel,storage,power,audio,video,docker -s /bin/bash pop
passwd pop

# Sudo for wheel group
echo "%wheel ALL=(ALL:ALL) ALL" > /etc/sudoers.d/wheel
```

### 5.3 Configure mkinitcpio

```bash
# Add btrfs hook to mkinitcpio (before filesystems)
perl -0777 -i.bak -pe '
  s{^(?!\s*#)\s*HOOKS=\(([^)]*)\)\s*$}{
    my @hooks = grep { length } split " ", $1;
    my %seen; @hooks = grep { !$seen{$_}++ } @hooks;
    my @out;
    for my $h (@hooks) {
      if ($h eq "filesystems") {
        push @out, "btrfs" unless $seen{btrfs}++;
      }
      push @out, $h;
    }
    "HOOKS=(" . join(" ", @out) . ")"
  }mge;
' /etc/mkinitcpio.conf

mkinitcpio -P
```

## GRUB Bootloader

🚀 Goal: install GRUB to the EFI System Partition and generate a `grub.cfg` that boots your Arch kernel/initramfs from Btrfs by UUID. GRUB will be installed to `/boot` (the FAT32 ESP), while kernel and initramfs files live on the Btrfs `/boot` subvolume.

### 6.1 Install GRUB packages

```bash

# Install GRUB (efibootmgr and dosfstools already in pacstrap)

pacman -S --noconfirm --needed grub

# Install GRUB to the EFI System Partition

grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB

```

### 6.2 Configure GRUB cmdline

📝 **Note:** Set up kernel parameters in `/etc/default/grub` for Btrfs root, resume (hibernation), and optional zswap support.

```bash

# Detect CPU and get UUIDs

ucode_img="intel"
lscpu | grep -qi amd && ucode_img="amd"

# Auto-detect swap partition (any disk)
swap_part="$(blkid -t TYPE=swap -o device | head -1)"
if [ -z "$swap_part" ]; then
  echo "ERROR: No swap partition found" >&2
  exit 1
fi
swap_uuid="$(blkid -s UUID -o value "$swap_part")"

# Remove any existing GRUB_CMDLINE_LINUX_DEFAULT line, then insert after GRUB_CMDLINE_LINUX=

sed -i '/^GRUB_CMDLINE_LINUX_DEFAULT=/d' /etc/default/grub
sed -i "/^GRUB_CMDLINE_LINUX=/a GRUB_CMDLINE_LINUX_DEFAULT=\"loglevel=3 resume=UUID=${swap_uuid} zswap.enabled=1 zswap.compressor=lz4 zswap.max_pool_percent=50 zswap.zpool=zsmalloc ${ucode_img}_iommu=on iommu=pt\"" /etc/default/grub

# Verify the change

grep "^GRUB_CMDLINE_LINUX" /etc/default/grub

```

### 6.3 Generate GRUB configuration

```bash

# Generate grub.cfg

grub-mkconfig -o /boot/grub/grub.cfg

```

## Services & QoL

⚙️ Goal: enable the essential background services you want on every boot (networking, bluetooth, etc.).

```bash

# Enable core services

systemctl enable NetworkManager
systemctl enable bluetooth
systemctl enable sshd
systemctl enable reflector.timer
systemctl enable fstrim.timer

```

## Desktop Stack

🖥️ Goal: install Niri compositor + Noctalia v5 shell + KDE apps — lean but fully functional desktop without the Plasma bloat.

> **What we DON'T install:** `plasma-desktop`, `plasma-workspace`, `kwin`, `plasmashell`, `plasma-login-manager`. Niri is the compositor, Noctalia is the shell, `greetd`+`noctalia-greeter` is the login manager.

### Base Wayland Stack

```bash
# xdg-desktop-portal : Portal framework (file picker, screen share, Flatpak integration)
# xdg-desktop-portal-kde : KDE backend for portals
# qt6-wayland : Qt6 Wayland platform plugin
# xorg-xwayland : Run X11 apps under Wayland
# xwayland-satellite : Alternative XWayland (multi-window support)

pacman -S --noconfirm --needed \
  xdg-desktop-portal \
  xdg-desktop-portal-kde \
  qt6-wayland \
  xorg-xwayland \
  xwayland-satellite
```

### Niri Compositor + Noctalia v5

```bash
# niri : Scrollable-tiling Wayland compositor (official repo)
pacman -S --noconfirm --needed niri
```

Then exit chroot to install AUR packages as user:

```bash
exit        # exit chroot
umount -R /mnt
reboot      # boot into new system with basic Wayland
```

After reboot, install yay (see [YAY Package Manager](#yay-package-manager)), then:

```bash
# 🔒 AUR — Noctalia v5 native C++ desktop shell
yay -S --noconfirm --needed noctalia-git

# 🔒 AUR — Noctalia greeter (login screen)
yay -S --noconfirm --needed noctalia-greeter

# 🔒 AUR — NiriMod: visual, interactive niri config GUI
# Repo: https://github.com/srinivasr/nirimod
yay -S --noconfirm --needed nirimod-git
```

### Display Manager (greetd + Noctalia Greeter)

```bash
# greetd : Generic greeter daemon (official repo)
sudo pacman -S --noconfirm --needed greetd

# Configure greetd to use Noctalia greeter
sudo tee /etc/greetd/config.toml <<'EOF'
[terminal]
vt = 1

[default_session]
command = "noctalia-greeter"
user = "greetd"
EOF

# Configure greeter settings
sudo mkdir -p /var/lib/noctalia-greeter
sudo tee /var/lib/noctalia-greeter/greeter.toml <<'EOF'
[session]
default = "niri-noctalia"

[keyboard]
layout = "us,th"
numlock = true

[cursor]
theme = "capitaine-cursors"
size = 24

[appearance]
password_style = "default"
EOF

# Enable greetd
sudo systemctl enable greetd
```

### Niri + Noctalia Session File

```bash
# Wrapper script
sudo tee /usr/local/bin/niri-noctalia-session << 'EOF'
#!/usr/bin/env bash
export XDG_CURRENT_DESKTOP=Noctalia
export XDG_SESSION_DESKTOP=noctalia
exec niri-session
EOF
sudo chmod +x /usr/local/bin/niri-noctalia-session

# Session .desktop file
sudo tee /usr/share/wayland-sessions/niri-noctalia.desktop <<'EOF'
[Desktop Entry]
Name=Niri + Noctalia v5
Comment=Session with Niri compositor and Noctalia v5 shell
Exec=/usr/local/bin/niri-noctalia-session
Type=Application
DesktopNames=Noctalia
EOF
```

### Desktop Apps (KDE + essentials)

```bash
# KDE desktp apps — same as Plasma but without the shell
pacman -S --noconfirm --needed \
  dolphin dolphin-plugins \
  kate okular gwenview spectacle ark \
  gparted kio-extras ffmpegthumbs kdegraphics-thumbnailers \
  filelight kcalc \
  btop fastfetch

# Terminal + sound
pacman -S --noconfirm --needed \
  ghostty libcanberra

# Cursor theme
pacman -S --noconfirm --needed capitaine-cursors
```

### Power Management

```bash
# power-profiles-daemon : Laptop power modes
pacman -S --noconfirm --needed power-profiles-daemon
systemctl enable power-profiles-daemon
```

### KDE Background Services

```bash
# KDE daemon + Qt platform theme for KDE apps
pacman -S --noconfirm --needed \
  plasma-integration \
  kded \
  qt6ct-kde
```

## Reboot

🔁 Goal: cleanly exit the installer environment and reboot into the new system.

```bash
exit

umount -R /mnt
swapoff -a
reboot
```

## Post-Install

Log in to your new system using your user account via the Noctalia greeter.

### Create XDG user directories

```bash

# Create standard user folders (Desktop, Documents, Downloads, etc.)

xdg-user-dirs-update

```

## SSH Setup

🔐 Goal: set up SSH key-based authentication for secure remote access to your new system.

### Generate SSH key pair (on your client machine)

```bash

# On your client machine (not the Arch system), generate an Ed25519 key

ssh-keygen -t ed25519 -C "your_email@example.com"

```

### Copy SSH key to Arch system

```bash

# From your client machine, copy the public key to your Arch system

ssh-copy-id -i ~/.ssh/id_ed25519.pub pop@arch

```

### SSH Security Hardening

🔒 Secure your SSH server by disabling password authentication and root login.

```bash

# Edit /etc/ssh/sshd_config

sudo tee -a /etc/ssh/sshd_config <<'EOF'

# SSH Security Settings

# Disable root login
PermitRootLogin no

# Disable password authentication (keys only)
PasswordAuthentication no

# Disable empty passwords
PermitEmptyPasswords no

# Disable keyboard-interactive authentication
KbdInteractiveAuthentication no

# Limit max authentication attempts
MaxAuthTries 3

# Set login grace time (60 seconds)
LoginGraceTime 60

# Enable strict mode
StrictModes yes

# Disable host-based authentication
HostbasedAuthentication no
IgnoreRhosts yes

# Set client alive interval for idle timeout (optional)
ClientAliveInterval 300
ClientAliveCountMax 2
EOF

# Restart SSH
sudo systemctl restart sshd

```

> ⚠️ **Test before disconnecting!** Open a new terminal and verify key-based login works before closing your current session.

## YAY Package Manager

```bash
sudo pacman -S --noconfirm --needed git base-devel && git clone https://aur.archlinux.org/yay.git && cd yay && makepkg -si
cd ..
rm -rf yay

```

### AUR Security Practices 🔒

⚠️ **Important:** The AUR is user-maintained and has had malware incidents (see [Arch Security Notice 2026](https://archlinux.org/news/active-aur-malicious-packages-incident/)). Always review PKGBUILDs before installing.

#### Review PKGBUILD before installing

When installing AUR packages, yay will show the PKGBUILD diff. **Always read it.** Look for:

- Suspicious URLs in `source=` (should point to official project repos/releases)
- Unexpected commands in `build()` or `package()` functions
- Curl/wget piping to bash or sh
- Obfuscated code or base64-encoded strings
- Network calls to unknown domains

#### Trust indicators

✅ **Safer:** Packages with many votes, active maintainer, links to official project  
⚠️ **Caution:** Orphaned packages, new maintainers, generic names  
❌ **Avoid:** Packages with no source URL, suspicious build scripts

This guide marks AUR packages clearly. Review them before running `yay -S`.

## Snapper

📸 Goal: set up snapshot management for Btrfs so you can roll back system changes.

### Official packages (pacman)

```bash
sudo pacman -S --noconfirm --needed snapper btrfs-assistant

```

### AUR packages (yay) 🔒

Review PKGBUILDs before installing. These are well-established packages but always verify.

```bash
yay -S --noconfirm --needed grub-btrfs snap-pac snap-pac-grub snapper-gui-git

```

### Enable grub-btrfs service

```bash

# grub-btrfs will auto-generate GRUB entries for snapshots

sudo systemctl enable --now grub-btrfsd

```

### Create Snapper Configs

📝 `/home` snapshotting is optional — skip if you have large media/game folders or prefer separate backup for user data.

```bash
sudo snapper -c root create-config /
sudo snapper -c boot create-config /boot
sudo snapper -c home create-config /home
# ↑ Skip /home if you don't want user data in snapshots

```

### Timeline & Retention Settings

```bash
# === root — hourly for quick undo ===
sudo snapper -c root set-config TIMELINE_CREATE=yes
sudo snapper -c root set-config TIMELINE_LIMIT_HOURLY=5
sudo snapper -c root set-config TIMELINE_LIMIT_DAILY=7
sudo snapper -c root set-config TIMELINE_LIMIT_WEEKLY=4
sudo snapper -c root set-config TIMELINE_LIMIT_MONTHLY=0
sudo snapper -c root set-config TIMELINE_LIMIT_YEARLY=0

# === boot — snap-pac covers kernel updates, no timeline needed ===
sudo snapper -c boot set-config TIMELINE_CREATE=no

# === home — light retention ===
sudo snapper -c home set-config TIMELINE_CREATE=yes
sudo snapper -c home set-config TIMELINE_LIMIT_HOURLY=0
sudo snapper -c home set-config TIMELINE_LIMIT_DAILY=3
sudo snapper -c home set-config TIMELINE_LIMIT_WEEKLY=2
sudo snapper -c home set-config TIMELINE_LIMIT_MONTHLY=0
sudo snapper -c home set-config TIMELINE_LIMIT_YEARLY=0
```

| Config | Hourly | Daily | Weekly | Logic |
|--------|--------|-------|--------|-------|
| `root` | 5 | 7 | 4 | System files, hourly for quick undo |
| `boot` | — | — | — | `TIMELINE_CREATE=no` — snap-pac covers kernel updates |
| `home` | 0 | 3 | 2 | User data, daily is enough |

### Number Limit (hard cap)

```bash
sudo snapper -c root set-config NUMBER_CLEANUP=yes
sudo snapper -c root set-config NUMBER_LIMIT=15
sudo snapper -c root set-config NUMBER_LIMIT_IMPORTANT=5
sudo snapper -c root set-config NUMBER_MIN_AGE=1800

sudo snapper -c boot set-config NUMBER_CLEANUP=yes
sudo snapper -c boot set-config NUMBER_LIMIT=5
sudo snapper -c boot set-config NUMBER_LIMIT_IMPORTANT=3
sudo snapper -c boot set-config NUMBER_MIN_AGE=1800

sudo snapper -c home set-config NUMBER_CLEANUP=yes
sudo snapper -c home set-config NUMBER_LIMIT=10
sudo snapper -c home set-config NUMBER_LIMIT_IMPORTANT=3
sudo snapper -c home set-config NUMBER_MIN_AGE=1800
```

### Enable Snapper Timers

```bash
sudo systemctl enable --now snapper-timeline.timer
sudo systemctl enable --now snapper-cleanup.timer
```

## Extra Packages

🧺 Personal "daily driver" pick list. Install only what you want — each section can be skipped.

### Core CLI + media (official)

```bash
sudo pacman -S --noconfirm --needed imagemagick vlc gimp obs-studio
```

### Filesystem / network / power (official)

```bash
sudo pacman -S --noconfirm --needed gvfs gvfs-smb brightnessctl
```

### JavaScript/TypeScript runtimes (official)

```bash
sudo pacman -S --noconfirm --needed nodejs npm bun
```

### Desktop apps (official)

```bash
sudo pacman -S --noconfirm --needed \
  firefox chromium libreoffice-fresh filezilla
```

### Desktop apps (AUR) 🔒

```bash
yay -S --noconfirm --needed \
  brave-bin \
  mailspring-bin \
  visual-studio-code-bin
```

### Gaming (official)

```bash
sudo pacman -S --noconfirm --needed \
  gamemode lib32-gamemode steam lutris \
  mangohud lib32-mangohud goverlay
sudo usermod -aG gamemode $USER
```

### Gaming (AUR) 🔒

```bash
yay -S --noconfirm --needed proton-ge-custom-bin
```

### Flatpak runtime (official)

```bash
sudo pacman -S --noconfirm --needed flatpak
```

### Fonts (official)

```bash
sudo pacman -S --noconfirm --needed \
  noto-fonts noto-fonts-emoji \
  ttf-dejavu ttf-ubuntu-font-family \
  terminus-font nerd-fonts ttf-ms-fonts
```

### Spell check (hunspell)

```bash
sudo pacman -S --noconfirm --needed hunspell hunspell-en_us hunspell-en_gb

# Thai dictionary (AUR) 🔒
yay -S --noconfirm --needed hunspell-th
```

## GPU Driver

🟩 Auto-detect your GPU and install the appropriate driver.

- **NVIDIA dGPU** → Installs `nvidia-open-dkms` + full userspace stack
- **Intel/AMD iGPU** → No extra driver needed (kernel drivers included in base install)

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

### NVIDIA dGPU (if detected)

```bash
if [ "$gpu_vendor" = "nvidia" ]; then
  echo "Installing NVIDIA open driver..."

  sudo pacman -S --noconfirm --needed \
    nvidia-open-dkms \
    nvidia-utils \
    lib32-nvidia-utils \
    nvidia-settings \
    libxnvctrl \
    ocl-icd \
    opencl-nvidia \
    lib32-opencl-nvidia \
    clinfo \
    cuda
fi
```

Gaming extras (NVIDIA only):

```bash
if [ "$gpu_vendor" = "nvidia" ]; then
  sudo pacman -S --noconfirm --needed \
    libva-utils \
    vdpauinfo \
    vulkan-tools \
    libva-nvidia-driver \
    dxvk \
    vkd3d \
    shaderc \
    spirv-tools
fi
```

### NVIDIA DRM kernel params + initramfs

Only run this section if an NVIDIA GPU was detected.

Add to `/etc/default/grub`:

```bash
# NVIDIA params added to GRUB_CMDLINE_LINUX_DEFAULT
nvidia-drm.modeset=1 nvidia-drm.fbdev=1
```

Configure mkinitcpio for NVIDIA:

```bash
if [ "$gpu_vendor" = "nvidia" ]; then
  sudo perl -0777 -i.bak -pe '
    s{^(?!\s*#)\s*MODULES=\(([^)]*)\)\s*$}{
      my @mods = grep { length } split " ", $1;
      my %seen; @mods = grep { !$seen{$_}++ } @mods;
      for my $add (qw(nvidia nvidia_modeset nvidia_uvm nvidia_drm)) {
        push @mods, $add unless $seen{$add}++;
      }
      "MODULES=(" . join(" ", @mods) . ")"
    }mge;

    s{^(?!\s*#)\s*HOOKS=\(([^)]*)\)\s*$}{
      my @hooks = grep { length && $_ ne "kms" } split " ", $1;
      "HOOKS=(" . join(" ", @hooks) . ")"
    }mge;
  ' /etc/mkinitcpio.conf

  sudo mkinitcpio -P
fi
```

NVIDIA pacman hook:

```bash
if [ "$gpu_vendor" = "nvidia" ]; then
  sudo mkdir -p /etc/pacman.d/hooks/

  sudo tee /etc/pacman.d/hooks/nvidia.hook >/dev/null <<'EOF'
[Trigger]
Operation = Install
Operation = Upgrade
Operation = Remove
Type = Package
Target = nvidia-open-dkms
Target = linux
Target = linux-zen

[Action]
Description = Update Nvidia module in initramfs
Depends = mkinitcpio
When = PostTransaction
NeedsTargets
Exec = /bin/sh -c 'while read -r trg; do case $trg in linux) exit 0; esac; done; /usr/bin/mkinitcpio -P'
EOF
fi
```

Regenerate GRUB (NVIDIA only):

```bash
if [ "$gpu_vendor" = "nvidia" ]; then
  grub-mkconfig -o /boot/grub/grub.cfg
fi
```

## KDE Integration

KDE apps (Dolphin, Kate, Okular) need background services for full functionality.

```bash
# KDE daemon — file dialogs, mime types, trash, KIO network
# Already installed in Desktop Stack section.
# Autostart it in Niri config:
```

Add to `~/.config/niri/config.kdl`:

```kdl
spawn-at-startup "noctalia"
spawn-at-startup "kded6"
```

### KWallet (Optional)

KWallet stores passwords for KDE apps (VS Code, Git credential helpers, browser passwords). On Niri+Noctalia with `greetd`, KWallet needs PAM config on greetd (not plasmalogin).

```bash
# kwallet : KDE wallet backend + secret service provider
# kwalletmanager : GUI to inspect/manage wallets
# kwallet-pam : unlocks the wallet at login (prevents repeated prompts)

sudo pacman -S --noconfirm --needed \
  kwallet \
  kwalletmanager \
  kwallet-pam
```

Then add KWallet PAM hook to greetd:

```bash
# Add pam_kwallet5 to greetd PAM (greetd is our display manager)
if ! grep -q 'pam_kwallet5' /etc/pam.d/greetd 2>/dev/null; then
  echo 'session    optional     pam_kwallet5.so auto_start' | sudo tee -a /etc/pam.d/greetd
fi
```

> **Auto-unlock requirements:** Wallet password must match login password, must use **blowfish** encryption (not GPG), wallet name must be `kdewallet`. See [KDE Wallet ArchWiki](https://wiki.archlinux.org/title/KDE_Wallet#Unlock_KDE_Wallet_automatically_on_login).

Or disable KWallet entirely if you don't use it:

```bash
mkdir -p ~/.config
cat <<'EOF' >> ~/.config/kwalletrc
[Wallet]
Enabled=false
EOF
```

## Niri + Noctalia Post-Config

📝 For detailed config (keybinds, wallpaper, blur, animations, gaming rules, greeter config), see **[Niri_Noctalia_v5.md](Niri_Noctalia_v5.md)** — the companion guide covering:

- Niri `config.kdl` with Noctalia IPC keybinds
- Wallpaper options (blurred overview / stationary / flat color)
- Blur & window appearance
- Complete DE Experience (animations, input QoL, gaming rules)
- Noctalia shell config (TOML)
- Greeter config reference

### Quick bootstrap config

Create `~/.config/niri/config.kdl`:

```kdl
input {
    keyboard { xkb { layout "us,th" options "grp:lalt_lshift_toggle" } }
}

spawn-at-startup "noctalia"
spawn-at-startup "kded6"

window-rule {
    geometry-corner-radius 20
    clip-to-geometry true
}

window-rule {
    match app-id="dev.noctalia.Noctalia"
    open-floating true
    default-column-width { fixed 1080; }
    default-window-height { fixed 920; }
}

debug { honor-xdg-activation-with-invalid-serial }

// Wallpaper — stationary
layer-rule { match namespace="^noctalia-wallpaper" place-within-backdrop true }
layout { background-color "transparent" }
overview { workspace-shadow { off } }

// Blur
window-rule { background-effect { blur true xray false } }
layer-rule { match namespace="^noctalia-(bar-[^"]+|notification|dock|panel|attached-panel|osd)$" background-effect { xray false } }
blur { passes 2 offset 3.0 noise 0.03 saturation 1.0 }

binds {
    Mod+Space   { spawn-sh "noctalia msg panel-toggle launcher"; }
    Mod+S       { spawn-sh "noctalia msg panel-toggle control-center"; }
    Mod+Comma   { spawn-sh "noctalia msg settings-toggle"; }
    XF86AudioRaiseVolume  { spawn-sh "noctalia msg volume-up; canberra-gtk-play -i audio-volume-change -d 'volume feedback'"; }
    XF86AudioLowerVolume  { spawn-sh "noctalia msg volume-down; canberra-gtk-play -i audio-volume-change -d 'volume feedback'"; }
    XF86AudioMute         { spawn-sh "noctalia msg volume-mute; canberra-gtk-play -i audio-volume-change -d 'volume feedback'"; }
    XF86MonBrightnessUp   { spawn-sh "noctalia msg brightness-up"; }
    XF86MonBrightnessDown { spawn-sh "noctalia msg brightness-down"; }
}
```

## pyenv

🐍 Goal: install `pyenv` so you can manage multiple Python versions per‑user.

```bash

# Install common build deps for compiling Python versions

sudo pacman -S --noconfirm --needed openssl zlib xz tk readline sqlite libffi bzip2

# Clone pyenv into your home directory

git clone https://github.com/pyenv/pyenv.git ~/.pyenv

```

### Shell integration (bash/zsh)

```bash
shell_name="$(basename "${SHELL:-bash}")"
conf="$HOME/.bashrc"
if [ "$shell_name" = "zsh" ]; then conf="$HOME/.zshrc"; fi
touch "$conf"

if ! grep -qF 'PYENV_ROOT' "$conf"; then
  cat <<'EOF' >> "$conf"

# pyenv

export PYENV_ROOT="$HOME/.pyenv"
export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init -)"
EOF
fi

```

### Install Python

```bash
pyenv install 3.13.2
pyenv global 3.13.2
python --version
```

## Flatpak Apps

Can't do this over SSH; install this in the logged-in session.

```bash
flatpak install -y flathub \
  io.github.jonmagon.kdiskmark \
  io.github.flattool.Warehouse \
  io.missioncenter.MissionCenter \
  xyz.z3ntu.razergenie \
  io.github.arunsivaramanneo.GPUViewer \
  com.kgurgul.cpuinfo \
  org.raspberrypi.rpi-imager \
  io.github.shonubot.Spruce \
  org.localsend.localsend_app \
  com.agateau.nanonote \
  org.gnome.Cheese \
  com.discordapp.Discord \
  com.cassidyjames.butler \
  net.davidotek.pupgui2 \
  com.vysp3r.ProtonPlus

```

## Theme

🎨 These are my personal favorites — use them as a starting point and change anything you don't like.

> **Note:** Noctalia v5 has its own theming system via Settings GUI + `~/.config/noctalia/config.toml`. The themes below are for KDE apps and cursors. SDDM themes are **not applicable** (we use `noctalia-greeter`).

### DE Themes

```bash

# Whitesur KDE (skip SDDM installs — not applicable)

repo=~/Desktop/theme
git clone https://github.com/vinceliuice/WhiteSur-kde.git $repo
cd $repo
./install.sh
# cd sddm && sudo ./install.sh  # SKIP — not using SDDM
rm -rf $repo && cd ~

# Orchis KDE
repo=~/Desktop/theme
git clone https://github.com/vinceliuice/Orchis-kde.git $repo
cd $repo
./install.sh
rm -rf $repo && cd ~

# Qogir KDE
repo=~/Desktop/theme
git clone https://github.com/vinceliuice/Qogir-kde.git $repo
cd $repo
./install.sh
rm -rf $repo && cd ~

# Vinyl Theme (AUR) 🔒
sudo pacman -S --noconfirm --needed python-cairosvg
yay -S --noconfirm --needed vinyl-git

```

### Icon themes & cursors

```bash

# Tela-icon-theme

repo=~/Desktop/icons
git clone https://github.com/vinceliuice/Tela-icon-theme.git $repo
cd $repo
./install.sh standard black blue nord
rm -rf $repo && cd ~

# Colloid-icon-theme

repo=~/Desktop/icons
git clone https://github.com/vinceliuice/Colloid-icon-theme.git $repo
cd $repo
./install.sh
cd cursor && sudo ./install.sh    # cursors
rm -rf $repo && cd ~

# Fluent-icon-theme

repo=~/Desktop/icons
git clone https://github.com/vinceliuice/Fluent-icon-theme.git $repo
cd $repo
./install.sh
cd cursor && sudo ./install.sh    # cursors
rm -rf $repo && cd ~

# Qogir-icon-theme

repo=~/Desktop/icons
git clone https://github.com/vinceliuice/Qogir-icon-theme.git $repo
cd $repo
./install.sh
rm -rf $repo && cd ~

```

### Wallpapers

```bash

# WhiteSur Wallpapers

repo=~/Desktop/wallpaper
git clone https://github.com/vinceliuice/WhiteSur-wallpapers.git $repo
cd $repo
./install-wallpapers.sh
rm -rf $repo && cd ~

```

## Credits & Thanks

🙏 Huge thanks to the authors/maintainers of these guides and notes that helped shape parts of this memo:

- `fstab` notes: <https://gist.github.com/mjkstra/96ce7a5689d753e7a6bdd92cdc169bae#fstab>
- Arch install walkthrough/reference: <https://github.com/silentz/arch-linux-install-guide>
- Chroot exit + reboot checklist: <https://gist.github.com/yovko/512326b904d120f3280c163abfbcb787#exit-from-chroot-and-reboot>
- NVIDIA driver guide: <https://github.com/korvahannu/arch-nvidia-drivers-installation-guide>
- Noctalia v5 docs: <https://docs.noctalia.dev/v5/>
- Niri Arch Wiki: <https://wiki.archlinux.org/title/Niri>
- CachyOS Niri config patterns: <https://github.com/CachyOS/cachyos-niri-noctalia>
