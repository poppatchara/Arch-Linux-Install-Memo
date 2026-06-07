# Arch Linux + Btrfs + CachyOS Kernels + KDE Plasma + Limine + Nvidia Driver

Personal notes for rebuilding my daily Arch install: UEFI firmware, single NVMe drive, Btrfs root, KDE Plasma desktop (with Plasma Login Manager), Limine bootloader, and ready for dual-booting, Nvidia DKMS, and Snapper-style snapshots.

Not the best way or most correct way. Just the way I like.

## Contents

1. [Updates](#updates)
2. [Assumptions](#assumptions)
3. [Live ISO Prep](#live-iso-prep)
4. [Partition & Format](#partition--format)
5. [Btrfs Subvolumes & Mounts](#btrfs-subvolumes--mounts)
6. [Base Install](#base-install)
7. [Chroot Configuration](#chroot-configuration)
8. [Limine Bootloader](#limine-bootloader)
9. [Services & QoL](#services--qol)
10. [Desktop Stack](#desktop-stack)
11. [Reboot](#reboot)
12. [Post-Install Ideas](#post-install-ideas)
    - [YAY package manager](#yay-package-manager)
    - [Snapper](#snapper)
    - [Extra Packages for Limine](#extra-packages-for-limine)
    - [CachyOS Kernels](#cachyos-kernels)
    - [CachyOS Packages](#cachyos-packages)
    - [Update all packages to CachyOS Optimized](#update-all-packages-to-cachyos-optimized)
    - [Extra Packages and fonts](#extra-packages-and-fonts)
    - [SPDIF audio dropout / sleep](#spdif-audio-dropout--sleep)
    - [GPU Driver (Automatic Detection)](#gpu-driver-automatic-detection)
    - [Disable KWallet prompts](#disable-kwallet-prompts-optional)
    - [Clear package manager caches](#clear-package-manager-caches)

13. [pyenv](#pyenv)
14. [Flatpak Apps](#flatpak-apps)
15. [Theme](#theme)
16. [Credits & Thanks](#credits--thanks)

## Updates

### 2026-03-21

- Created a GRUB-based variant: [Arch Linux_Zen_Grub_Brtfs.md](Arch Linux_Zen_Grub_Brtfs.md). Uses `linux-zen`, GRUB bootloader, and Btrfs `/boot` subvolume for snapshots. No CachyOS repos.

### 2026-06-07

- GPU: Auto-detect vendor (NVIDIA/Intel/AMD) via `lspci`. Only install NVIDIA driver if dGPU present; iGPU systems skip automatically.
- NVIDIA: Replaced proprietary Option B with single `nvidia-open-dkms` (610.x). All NVIDIA blocks wrapped in `if [ "$gpu_vendor" = "nvidia" ]`.
- Desktop: Replaced SDDM with `plasma-login-manager` (Plasma 6.6+ native login manager). Added SDDM cleanup section.
- Typo: `fatfetch` → `fastfetch`.
- Python: Bumped pyenv install from 3.12 → 3.13.2.

### 2025-12-26

- Narrowed the KDE stutter down to themes: Whitesur, Orchis, and Colloid caused the issue on my machine. Qogir, Darkly, and Vinyl do not stutter here. This may vary by hardware, so test on your setup.

### 2025-12-22

- I'm going back to `linux-zen` (and may try `xanmod` or `tkg`). I noticed a very small but annoying stutter in KDE Plasma when resizing windows; it may have been present on the vanilla kernel too and I just never noticed it.
- I'll keep the CachyOS section for now in case anyone finds it useful. I'll come back with my findings after more testing.

## Assumptions

🧩 This memo is written as a linear “do this, then that” install. If you deviate (different disk name, multiple disks, LUKS, existing Windows ESP, etc.), adjust the affected commands and double-check mounts/UUIDs before continuing.

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
sudo cp "$mirrorfile" "${mirrorfile}.bak"

tmpfile="$(mktemp)"
sudo awk -v country="$country" '
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
' "$mirrorfile" > "$tmpfile" || { echo "mirrorlist reorder failed; restoring backup." >&2; sudo cp "${mirrorfile}.bak" "$mirrorfile"; rm -f "$tmpfile"; exit 1; }

sudo cp "$tmpfile" "$mirrorfile"
rm -f "$tmpfile"

sudo pacman -Syy

```

## Partition & Format

💽 Create a minimal GPT layout (ESP + Btrfs root + swap) and capture stable UUIDs for later bootloader and `fstab` configuration. Before running format commands, sanity-check your target disk with `lsblk -f` so you don’t wipe the wrong device.

### 1. My Partition layout

| Partition | Size | Type | Purpose |
|-----------|------|------|---------|
| `/dev/nvme0n1p1` | 2-4 GB | EFI System (type `ef00`) | Mounted at `/boot` |
| `/dev/nvme0n1p2` | Remainder | Linux filesystem (`8300`) | Btrfs root |
| `/dev/nvme0n1p3` | About half or equal your RAM size | Linux swap | Swap |

You can use `cfdisk /dev/nvme0n1` (GPT) to build or adjust the layout.

### 2. Format and capture UUIDs

**CAUTION:** double-check your device paths (e.g. `/dev/nvme0n1p2`) before formatting.

```bash

# Change the disk path to match yours!!!

mkfs.btrfs -f -L ArchLinuxFS /dev/nvme0n1p2
mkswap /dev/nvme0n1p3

# Store UUID for later scripts.

esp_uuid="$(blkid -s UUID -o value /dev/nvme0n1p1)"
root_uuid="$(blkid -s UUID -o value /dev/nvme0n1p2)"
swap_uuid="$(blkid -s UUID -o value /dev/nvme0n1p3)"

```

## Btrfs Subvolumes & Mounts

🧱 Goal: create a subvolume layout that plays nicely with snapshot tools (e.g., Snapper) and keeps “churny” paths isolated (logs/cache). This is a personal layout; feel free to add/remove subvolumes depending on what you plan to snapshot.

📝 Notes:

- Subvolumes here are named `@something` by convention; `@` is mounted as `/` later.
- `compress=zstd:1,noatime` is a common baseline; tune for your workload.
- If you don’t care about isolating `/var/log` or `/var/cache`, you can skip those subvolumes and mounts.
- I have Downloads, `.cache`, and Git isolated. These usually get big and I don’t want to snapshot them.

### 3.1 Create Subvolumes and Mounts

```bash

# Create subvolumes

mount UUID="${root_uuid}" /mnt

btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@var
btrfs subvolume create /mnt/@var_log
btrfs subvolume create /mnt/@var_cache
btrfs subvolume create /mnt/@root
btrfs subvolume create /mnt/@srv

# OPTIONAL: extra subvolumes

# I find these folders usually grow large on my system and they change quite often, so I opted not to snapshot them.

# You can skip these. Especially `home/Git` — that’s my place for all git repos.

#btrfs subvolume create /mnt/@home_cache

#btrfs subvolume create /mnt/@home_downloads
#btrfs subvolume create /mnt/@home_git

# remount as btrfs subvolumes

umount -R /mnt
mount -o compress=zstd:1,noatime,subvol=@ UUID="${root_uuid}" /mnt

mount --mkdir -o compress=zstd:1,noatime,subvol=@home UUID="${root_uuid}" /mnt/home
mount --mkdir -o compress=zstd:1,noatime,subvol=@var  UUID="${root_uuid}" /mnt/var
mount --mkdir -o compress=zstd:1,noatime,subvol=@var_log  UUID="${root_uuid}" /mnt/var/log
mount --mkdir -o compress=zstd:1,noatime,subvol=@var_cache UUID="${root_uuid}" /mnt/var/cache
mount --mkdir -o compress=zstd:1,noatime,subvol=@root UUID="${root_uuid}" /mnt/root
mount --mkdir -o compress=zstd:1,noatime,subvol=@srv  UUID="${root_uuid}" /mnt/srv

# If you created optional subvolumes, mount them.

# I'm the only user of this machine, so I will mount it directly to my home.

# You will need to come up with symlink solution for multiusers.

#mkdir -p /mnt/mnt/homes

#mount --mkdir -o compress=zstd:1,noatime,subvol=@home_cache UUID="${root_uuid}" /mnt/home/pop/.cache
#mount --mkdir -o compress=zstd:1,noatime,subvol=@home_downloads UUID="${root_uuid}" /mnt/home/pop/Downloads

#mount --mkdir -o compress=zstd:1,noatime,subvol=@home_git UUID="${root_uuid}" /mnt/home/pop/Git

```

### 3.2 Swap, ESP and fstab

```bash

# Mount ESP at /boot

mount --mkdir UUID="${esp_uuid}" /mnt/boot

# Enable swap

swapon UUID="${swap_uuid}"

# Generate fstab

mkdir -p /mnt/etc
genfstab -U /mnt > /mnt/etc/fstab
cat /mnt/etc/fstab

```

## Base Install

📦 Goal: install a bootable base system onto `/mnt` (kernel, firmware, networking, editors, etc.), then switch into it with `arch-chroot`. If you prefer fewer packages, trim this list, but keep `base`, a kernel, `linux-firmware`, and whatever you need for networking and your filesystem.

### 4.1 Install base packages

```bash

# Detect cpu

cpu=intel
if lscpu | grep -qi amd; then
  cpu=amd
fi

# Prepare vconsole

# ter-116n, ter-120n, ter-124n, ter-32n

# or use any others you like

cat <<'EOF' > /mnt/etc/vconsole.conf
KEYMAP=us
FONT=ter-124n
EOF

# Install packages

pacstrap -K /mnt \
  base base-devel \
  linux linux-headers linux-firmware "${cpu}-ucode" \
  efibootmgr btrfs-progs dosfstools e2fsprogs exfatprogs\
  networkmanager openssh \
  vim nvim git sudo man curl wget perl \
  zsh zsh-completions zsh-autosuggestions bash-completion \
  pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber reflector \
  inotify-tools

# copy pacman config

cp /etc/pacman.conf /mnt/etc/pacman.conf

```

<details>
  <summary>📦 Packages being installed (Base Install)</summary>

- Core system: `base`, `base-devel`
- Kernel + firmware: `linux`, `linux-headers`, `linux-firmware`
- CPU microcode: `intel-ucode` or `amd-ucode` (chosen via `lscpu`)
- Filesystem tools: `btrfs-progs`
- Boot/UEFI tools: `efibootmgr`
- Networking + SSH: `networkmanager`, `openssh`
- Editors + tools: `vim`, `nvim`, `git`, `sudo`, `man`, `curl`, `wget`, `perl`, `inotify-tools`
- Audio stack (PipeWire): `pipewire`, `pipewire-alsa`, `pipewire-pulse`, `pipewire-jack`, `wireplumber`
- Mirrors/shell UX: `reflector`, `zsh`, `zsh-completions`, `zsh-autosuggestions`, `bash-completion`

</details>

### 4.2 Then chroot into our setup

```bash
arch-chroot /mnt

```

## Chroot Configuration

🏠 Everything below runs inside `arch-chroot /mnt` unless explicitly stated otherwise. This section sets the system identity (locale/time/hostname/users) and ensures your initramfs includes the bits needed to boot from Btrfs.

### 5.1 Locale, time, and console

```bash

# Change the timezone to yours.

ln -sf /usr/share/zoneinfo/Asia/Bangkok /etc/localtime
hwclock --systohc

# Uncomment desired locales in `/etc/locale.gen` (add or remove lines as appropriate for your setup).

# Japanese

sed -i 's/^#ja_JP.UTF-8 UTF-8/ja_JP.UTF-8 UTF-8/' /etc/locale.gen

# Thai

sed -i 's/^#th_TH.UTF-8 UTF-8/th_TH.UTF-8 UTF-8/' /etc/locale.gen

# British English

sed -i 's/^#en_GB.UTF-8 UTF-8/en_GB.UTF-8 UTF-8/' /etc/locale.gen

# US English

sed -i 's/^#en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen

locale-gen
echo 'LANG=en_US.UTF-8' > /etc/locale.conf

```

### 5.2 Hostname and hosts file

Change `host_name` to your liking.

```bash
host_name="arch"
echo "${host_name}" > /etc/hostname
cat <<EOF > /etc/hosts
127.0.0.1   localhost
::1         localhost
127.0.1.1   ${host_name}.localdomain ${host_name}
EOF

```

### 5.3 Users and sudo

**Root password**

```bash
echo "Setting Root Password"
passwd

```

**Create user and password**

```bash

# change username

user=pop
echo "Create user, and set password"
useradd -m -G wheel,storage,power,audio,video -s /bin/bash $user
passwd $user

```

```bash

# If you created optional subvolumes earlier, take ownership of the home.

chown -R pop:pop /home/pop

```

**Enable wheel sudo in `sudoers`**
We won’t automate this.

```bash

# Open and uncomment: %wheel ALL=(ALL) ALL

EDITOR=vim visudo

```

### 5.4 mkinitcpio

Set `MODULES`, `BINARIES`, and `HOOKS` for a basic Btrfs initramfs. Enable resume. Adjust the values to your own needs.

```bash
perl -pi -e '
  # Ensure btrfs module and `btrfs` binary are included in the initramfs
  s/^MODULES=\(\)/MODULES=(btrfs)/;
  s/^BINARIES=\(\)/BINARIES=(\/usr\/bin\/btrfs)/;

  # Replace the HOOKS= line with a more complete set (tune for your setup)
  s/^HOOKS=\(.*\)$/HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block encrypt filesystems resume fsck)/;
' /etc/mkinitcpio.conf

# Rebuild initramfs with the new mkinitcpio configuration

mkinitcpio -P

```

## Limine Bootloader

🚀 Goal: install Limine to the EFI System Partition and generate a `limine.conf` that boots your Arch kernel/initramfs from Btrfs by UUID. The key idea is to keep Limine’s config and the kernel artifacts together under `/boot/limine`.

### 6.1 Install Limine binaries (ESP mounted at `/boot`)

```bash

# Limine Bootloader

pacman -S --needed limine
mkdir -p /boot/EFI/limine /boot/limine
cp -v /usr/share/limine/*.EFI /boot/EFI/limine/

```

Some extra for Limine are in AUR, so we will install them later, after we got yay installed.

<details>
  <summary>🚀 Packages being installed (Limine)</summary>

- `limine`: Limine UEFI bootloader EFI binaries + templates/config support

</details>

### 6.2 Register Limine with the firmware

```bash
efibootmgr --create --disk /dev/nvme0n1 --part 1 \
  --label "Limine Bootloader" \
  --loader '\EFI\limine\BOOTX64.EFI' \
  --unicode

```

- `--disk /dev/nvme0n1 --part 1` corresponds to `/dev/nvme0n1p1`.
- Limine interprets `boot():/` as “the partition containing `limine.conf`”.

### 6.3 Copy kernels, initramfs, and microcode into `/boot/limine`

```bash
ucode_img="intel-ucode"
if lscpu | grep -qi amd; then
  ucode_img="amd-ucode"
fi

cp -v /boot/vmlinuz-linux /boot/limine/
cp -v /boot/initramfs-linux*.img /boot/limine/
cp -v "/boot/${ucode_img}.img" /boot/limine/

```

Repeat after every kernel, initramfs, or microcode update (consider adding a pacman hook later).

### 6.4 Generate `/boot/limine/limine.conf`

```bash
root_uuid="$(blkid -s UUID -o value /dev/nvme0n1p2)"
swap_uuid="$(blkid -s UUID -o value /dev/nvme0n1p3)"
ucode_img="intel"
if lscpu | grep -qi amd; then
  ucode_img="amd"
fi

cat <<EOF | tee /boot/limine/limine.conf >/dev/null
TIMEOUT=3
DEFAULT_ENTRY=Arch Linux

/Arch Linux
    PROTOCOL: linux
    KERNEL_PATH: boot():/limine/vmlinuz-linux
    MODULE_PATH: boot():/limine/${ucode_img}-ucode.img
    MODULE_PATH: boot():/limine/initramfs-linux.img
    CMDLINE: loglevel=3 root=UUID=${root_uuid} rootflags=subvol=@ rootfstype=btrfs rw resume=UUID=${swap_uuid} zswap.enabled=1 zswap.compressor=lz4 zswap.max_pool_percent=50 zswap.zpool=zsmalloc ${ucode_img}_iommu=on iommu=pt

/Arch Linux (fallback)
    PROTOCOL: linux
    KERNEL_PATH: boot():/limine/vmlinuz-linux
    MODULE_PATH: boot():/limine/initramfs-linux.img
    CMDLINE: loglevel=3 root=UUID=${root_uuid} rootflags=subvol=@ rootfstype=btrfs rw
EOF

cat /boot/limine/limine.conf

```

#### Add zswap setting to initrd (optional)

```bash

# Add zswap modules to mkinitcpio.conf (idempotent)

sudo perl -0777 -i.bak -pe '
  s{^(?!\s*#)\s*MODULES=\(([^)]*)\)\s*$}{
    my @mods = grep { length } split " ", $1;
    my %seen; @mods = grep { !$seen{$_}++ } @mods;
    for my $add (qw(lz4 lz4_compress zsmalloc)) {
      push @mods, $add unless $seen{$add}++;
    }
    "MODULES=(" . join(" ", @mods) . ")"
  }mge;
' /etc/mkinitcpio.conf
sudo mkinitcpio -P
grep -E '^MODULES=' /etc/mkinitcpio.conf

```

### 6.5 Pacman hook to redeploy Limine EFI files

```bash
sudo mkdir -p /etc/pacman.d/hooks
sudo tee /etc/pacman.d/hooks/99-limine.hook >/dev/null <<'EOF'
[Trigger]
Operation = Install
Operation = Upgrade
Type      = Package
Target    = limine

[Action]
Description = Copy Limine EFI files to the ESP
When = PostTransaction
Exec = /usr/bin/cp /usr/share/limine/BOOTX64.EFI /boot/EFI/limine/
EOF

```

## Services & QoL

⚙️ Goal: enable the essential background services you want on every boot (networking, printing, bluetooth, etc.). Pick a networking stack and enable only what you actually use to avoid service conflicts.

🧠 Networking note (choose one approach):

- `NetworkManager` (common on desktops/laptops): enable `NetworkManager` (and optionally `iwd` for Wi‑Fi backend).

### Extra packages (optional)

```bash

# core system tools & CLI utilities: util-linux inetutils usbutils rsync htop bat zip unzip p7zip

# iwd : wifi

# mDNS: avahi nss-mdns

# audio tools & firmware: alsa-utils sof-firmware easyeffects

# Bluetooth support: bluez bluez-utils

# cups : printing system

# power & hardware management: acpi acpi_call acpid

# xdg-user-dirs : user directory structure

pacman -Syu --needed \
  util-linux inetutils usbutils rsync htop bat zip unzip p7zip \
  iwd \
  avahi nss-mdns \
  alsa-utils sof-firmware easyeffects \
  bluez bluez-utils \
  cups \
  acpi acpi_call acpid \
  xdg-user-dirs

```

<details>
  <summary>⚙️ Packages being installed (Services &amp; QoL extras)</summary>

- Basics: `htop`, `inetutils`, `usbutils`, `rsync`, `util-linux`
- Archives: `zip`, `unzip`, `p7zip`
- CLI quality-of-life: `bat`
- Networking (pick what you use): `iwd`, `avahi`, `nss-mdns`
- Audio firmware/tools: `sof-firmware`, `alsa-utils`, `easyeffects`
- Bluetooth: `bluez`, `bluez-utils`
- Printing: `cups`
- Power/ACPI: `acpi`, `acpi_call`, `acpid`
- Desktop/user dirs: `xdg-user-dirs`

</details>

### Enable services

```bash

# Networking

systemctl enable NetworkManager

# Optional Wi‑Fi tooling (only enable if you actually use it)

# systemctl enable iwd

# Everything else (pick what you need)

systemctl enable bluetooth

# systemctl enable cups

# systemctl enable avahi-daemon

# systemctl enable acpid

systemctl enable reflector.timer
systemctl enable sshd
systemctl enable fstrim.timer

```

## Desktop Stack

🖥️ Goal: install KDE Plasma + Plasma Login Manager and the integration pieces you'll want on a typical desktop (portals, NetworkManager applet, audio, thumbnails). If you don't want many of the KDE apps, you can skip the Desktop Apps section.

### KDE Plasma & apps

#### KDE Core

```bash
# plasma-login-manager : KDE native login manager (Plasma 6.6+, replaces SDDM)
# xdg-desktop-portal : "Portal" framework (file picker, screen share, sandbox app integration)
# xdg-desktop-portal-kde : KDE backend for portals (needed for Wayland screen share, Flatpak, etc.)
# qt6-wayland : Qt6 Wayland platform plugin
# xorg-xwayland : Runs X11 apps under Wayland

pacman -S --needed \
  plasma-login-manager \
  xdg-desktop-portal \
  xdg-desktop-portal-kde \
  qt6-wayland \
  xorg-xwayland

systemctl enable plasmalogin
```

#### KDE Plasma Core

```bash
# plasma-desktop : The Plasma desktop shell (panels, launcher, desktop UI)
# plasma-workspace : Core workspace components (session bits, shell integration, essentials)
# kwin : KDE window manager + compositor (Wayland/X11)
# systemsettings : KDE System Settings app
# plasma-nm : NetworkManager integration (network tray, VPN UI)
# plasma-pa : Audio volume controls for PipeWire/PulseAudio
# kscreen : Display configuration + monitor hotplug handling
# kde-gtk-config : Configure GTK theme/fonts under KDE
# breeze-gtk : Breeze theme for GTK apps (visual consistency)

pacman -S --needed \
  plasma-desktop \
  plasma-workspace \
  kwin \
  systemsettings \
  plasma-nm \
  plasma-pa \
  kscreen \
  kde-gtk-config \
  breeze-gtk
```

#### KDE Plasma (Optionals)

You can select what you need.

```bash

# bluedevil : Bluetooth tray + pairing UI

# power-profiles-daemon : Laptop power modes (balanced/performance/powersave)

# kdeplasma-addons : Extra Plasma widgets/applets (more features, more stuff)

# plasma-systemmonitor : KDE System Monitor app (optional if you use htop/Mission Center)

# plasma-browser-integration : Browser media controls + integration

# discover : KDE software center (can add background notifier)

# krdp : KDE Remote Desktop server/client bits (krdpserver)

# print-manager : KDE printer management UI

# appmenu-gtk-module : AppMenu GTK module

# libdbusmenu-glib : DBus menu integration

pacman -S --needed \
  bluedevil \
  power-profiles-daemon \
  kdeplasma-addons \
  plasma-systemmonitor \
  plasma-browser-integration \
  discover \
  krdp \
  print-manager \
  appmenu-gtk-module \
  libdbusmenu-glib

systemctl enable power-profiles-daemon

# Optional: Encrypted “vault” folders integration: plasma-vault

# pacman -S --needed plasma-vault

# Optional: Phone integration (kdeconnectd service): kdeconnect

# pacman -S --needed kdeconnect

```

#### KWallet (optional, for secret storage like VS Code)

```bash
# kwallet : KDE wallet backend + secret service provider
# kwalletmanager : GUI to inspect/manage wallets
# kwallet-pam : unlocks the wallet at login (prevents repeated prompts)

pacman -S --needed \
  kwallet \
  kwalletmanager \
  kwallet-pam

# Add PAM hooks for Plasma Login Manager so the wallet unlocks with your session.

if ! grep -q pam_kwallet5 /etc/pam.d/plasmalogin; then
  sudo tee -a /etc/pam.d/plasmalogin >/dev/null <<'EOF'
auth       optional pam_kwallet5.so
session    optional pam_kwallet5.so auto_start
EOF
fi
```

- After logging in, open System Settings → KDE Wallet and ensure a wallet exists (Blowfish is fine). This keeps VS Code and other apps from nagging when storing secrets.

#### Clean up SDDM (post-migration)

After confirming Plasma Login Manager works, remove SDDM and its user:

```bash
sudo systemctl disable sddm
sudo systemctl enable plasmalogin

# Remove SDDM package and user
sudo pacman -Rns sddm sddm-kcm
sudo userdel -r sddm
```

#### Desktop Apps

````bash

# dolphin : File manager

# dolphin-plugins : Git/Share/extra Dolphin integrations

# konsole : Terminal

# kate : GUI text editor

# okular : PDF/EPUB viewer

# gwenview : Image viewer

# spectacle : Screenshot tool

# ark : Archive manager GUI

# gparted : Partition editor GUI

# kio-extras : SMB/SFTP/etc. support inside Dolphin/KIO

# ffmpegthumbs : Video thumbnails in Dolphin

# kdegraphics-thumbnailers : Document/image thumbnail plugins

# filelight : Disk usage GUI

# kcalc : Calculator

pacman -S --needed \
  dolphin \
  dolphin-plugins \
  konsole \
  kate \
  okular \
  gwenview \
  spectacle \
  ark \
  gparted \
  kio-extras \
  ffmpegthumbs \
  kdegraphics-thumbnailers \
  filelight \
  kcalc

````

<details>
  <summary>🖥️ Packages being installed (KDE Plasma desktop)</summary>

- Display manager: `plasma-login-manager`
- Portals/Wayland helpers: `xdg-desktop-portal`, `xdg-desktop-portal-kde`, `qt6-wayland`, `xorg-xwayland`
- Plasma core: `plasma-desktop`, `plasma-workspace` (includes Wayland session), `kwin`, `systemsettings`
- Plasma integration: `plasma-nm`, `plasma-pa`, `kscreen`, `kde-gtk-config`, `breeze-gtk`
- Optional Plasma extras: `bluedevil`, `power-profiles-daemon`, `kdeplasma-addons`, `plasma-systemmonitor`, `plasma-browser-integration`, `discover`, `krdp`, `print-manager`
- Desktop apps: `dolphin`, `dolphin-plugins`, `konsole`, `kate`, `okular`, `gwenview`, `spectacle`, `ark`, `gparted`, `filelight`, `kcalc`
- Thumbnails/integration: `kio-extras`, `ffmpegthumbs`, `kdegraphics-thumbnailers`

</details>

## Reboot

🔁 Goal: cleanly exit the installer environment and reboot into the new system. Before rebooting, it’s worth quickly checking `/mnt/etc/fstab` and that `/mnt/boot/limine/limine.conf` exists.

### Exit chroot and reboot

```bash
exit

```

Then:

```bash
umount -R /mnt
swapoff -a
reboot

```

## Post-Install Ideas

Log in to your new system using your user account.

### YAY package manager

```bash
sudo pacman -S --needed git base-devel && git clone https://aur.archlinux.org/yay.git && cd yay && makepkg -si
cd ..
rm -rf yay

```

<details>
  <summary>🧰 Packages being installed (bootstrap AUR helper)</summary>

- `git`: fetch the `yay` PKGBUILD repo
- `base-devel`: required toolchain for building AUR packages via `makepkg`

</details>

### Snapper

📸 Goal: set up snapshot management for Btrfs so you can roll back system changes. `snapper` itself is in the official repos; the Limine integration shown below uses AUR packages (requires an AUR helper such as `yay`, installed above).

```bash
sudo pacman -Syu snapper

```

<details>
  <summary>📸 Packages being installed (Snapper + optional Limine integration)</summary>

- `snapper`: manage Btrfs snapshots (create/list/rollback policies)
- `snap-pac`: pacman hooks to auto-create snapshots around package transactions
- (AUR) `limine-snapper-sync`: keep Limine config/entries aligned with snapshots
- (AUR) `limine-mkinitcpio-hook`: mkinitcpio hook for Limine workflows

</details>

#### Create Snapper Configs

You can omit `/home` snapping if you prefer (e.g., for large media folders).

```bash

# We need to be root for this part

sudo su

```

```bash
snapper -c root create-config /
snapper -c home create-config /home

```

```bash

# Exit root

exit

```

### Extra Packages for Limine

```bash

# Optional (AUR): keep Limine + initramfs in sync with snapshots/updates

yay -S \
  limine-snapper-sync \
  limine-mkinitcpio-hook \
  snap-pac

cp /etc/limine-snapper-sync.conf /etc/default/limine

# trigger sync

sudo limine-snapper-sync

```

```bash
yay -S --needed \
  btrfs-assistant \
  snapper-gui-git \
  snapper-tools

```

### CachyOS Kernels

🐆 If you want CachyOS’ tuned kernels/userspace, add their repo and install the kernel packages. This is optional and changes your system away from “pure Arch”, so it’s a good idea to keep a package list backup first.

Backup pacman config

```bash
sudo cp -a /etc/pacman.conf /etc/pacman.conf.pre-cachy
pacman -Qqe > ~/pkglist.pre-cachy.txt

```

Add CachyOS repos (automated way)

```bash
cd ~
curl https://mirror.cachyos.org/cachyos-repo.tar.xz -o cachyos-repo.tar.xz
tar xvf cachyos-repo.tar.xz && cd cachyos-repo
sudo ./cachyos-repo.sh

sudo pacman -Syu

```

Install CachyOS kernels

```bash

# CachyOS EEVDF kernel + headers

# Default CachyOS kernel + headers

sudo pacman -S \
  linux-cachyos-eevdf linux-cachyos-eevdf-headers \
  linux-cachyos linux-cachyos-headers

```

<details>
  <summary>🐆 Packages being installed (CachyOS kernels)</summary>

- `linux-cachyos`, `linux-cachyos-headers`: CachyOS kernel + headers
- `linux-cachyos-eevdf`, `linux-cachyos-eevdf-headers`: alternative CachyOS kernel variant + headers

</details>

### CachyOS Packages

```bash

# cachyos-settings : CachyOS defaults/tweaks

# appmenu-gtk-module : AppMenu GTK module

# libdbusmenu-glib : DBus menu integration

# cachyos-gaming-meta : Gaming-oriented meta package

# cachyos-hello : Welcome/info app

sudo pacman -S \
  cachyos-settings \
  appmenu-gtk-module \
  libdbusmenu-glib \
  cachyos-gaming-meta \
  cachyos-hello

```

<details>
  <summary>🐆 Packages being installed (CachyOS extras)</summary>

- `cachyos-settings`: CachyOS defaults/tweaks
- `cachyos-gaming-meta`: gaming-oriented meta package
- `cachyos-hello`: welcome/info app
- `appmenu-gtk-module`, `libdbusmenu-glib`: appmenu/DBus menu integration

</details>

### Update all packages to CachyOS Optimized

```bash
sudo pacman -Qqn | sudo pacman -S -

```

### Extra Packages and fonts

🧺 Personal “daily driver” pick list. Install only what you want.

#### Core CLI + utilities

```bash
yay -S --needed openbsd-netcat imagemagick

```

#### Filesystem / network integration

```bash
yay -S --needed gvfs gvfs-smb

```

#### Power / laptop bits

```bash
yay -S --needed brightnessctl

```

#### Audio / media / creative

```bash
yay -S --needed vlc gimp obs-studio

```

#### Desktop apps

```bash
yay -S --needed \
  firefox chromium brave-bin \
  libreoffice-fresh \
  mailspring-bin \
  visual-studio-code-bin \
  filezilla \
  fastfetch

```

#### Gaming stack

```bash
yay -S --needed \
  gamemode lib32-gamemode \
  steam \
  lutris \
  mangohud lib32-mangohud \
  goverlay \
  proton-ge-custom-bin
sudo usermod -aG gamemode $USER

```

#### Flatpak runtime

```bash
yay -S flatpak

```

#### Fonts

```bash
yay -S --needed \
  noto-fonts \
  noto-fonts-cjk \
  noto-fonts-emoji \
  noto-fonts-extra \
  ttf-dejavu \
  ttf-liberation \
  ttf-jetbrains-mono \
  ttf-fira-code \
  ttf-ubuntu-font-family \
  terminus-font \
  adobe-source-sans-fonts \
  adobe-source-serif-fonts \
  adobe-source-code-pro-fonts \
  nerd-fonts \
  ttf-ms-fonts

```

<details>
  <summary>🧺 Packages being installed (daily-driver pick list)</summary>

- Core utilities/tools: `openbsd-netcat`, `imagemagick`
- Desktop integration: `gvfs`, `gvfs-smb`
- Browsers/mail: `firefox`, `mailspring`
- Media/creative: `vlc`, `gimp`, `obs-studio`
- Gaming: `steam`, `lutris`, `gamemode`, `mangohud`, `goverlay`, `proton-ge-custom-bin`
- System/power: `brightnessctl`
- Packaging/platforms: `flatpak`
- Editors/IDE: `visual-studio-code-bin`
- Office: `libreoffice-fresh`
- Fonts: `noto-fonts`, `noto-fonts-cjk`, `noto-fonts-emoji`, `noto-fonts-extra`, `ttf-dejavu`, `ttf-liberation`, `ttf-jetbrains-mono`, `ttf-fira-code`, `ttf-ubuntu-font-family`, `terminus-font`, `adobe-source-sans-fonts`, `adobe-source-serif-fonts`, `adobe-source-code-pro-fonts`, `nerd-fonts`, `ttf-ms-fonts`

</details>

### SPDIF audio dropout / sleep

🔊 Some SPDIF receivers/DACs go to sleep after a few idle seconds, so the first 1-3 seconds of the next sound get eaten while the link wakes up. Two tweaks that keep the SPDIF clock alive:

1) Disable codec autosuspend (system-wide)

```bash
sudo tee /etc/modprobe.d/alsa-no-powersave.conf >/dev/null <<'EOF'
options snd_hda_intel power_save=0 power_save_controller=N
EOF

```

Reboot for this to take effect.

1) Stop WirePlumber from suspending the sink (per-user)

```bash
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

Change the match if you only want this on a specific output. With autosuspend off and WirePlumber keeping the sink alive, SPDIF should stop dropping the first few seconds of audio.

### GPU Driver (Automatic Detection)

🟩 Arch Linux now defaults to open kernel modules for NVIDIA. This section auto-detects your GPU and installs the appropriate driver.

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

#### NVIDIA dGPU (if detected)

> **Note:** `nvidia-open-dkms` is recommended for CachyOS and other non-vanilla kernels. For the vanilla `linux` kernel, use `nvidia-open` instead. GSP firmware cannot be disabled in the open driver (see Arch Wiki).

```bash
if [ "$gpu_vendor" = "nvidia" ]; then
  echo "Installing NVIDIA open driver..."

  # nvidia-open-dkms : NVIDIA open kernel modules - module sources (DKMS)
  # nvidia-utils : NVIDIA userspace libraries + tools
  # lib32-nvidia-utils : 32-bit NVIDIA libs (Steam/Proton)
  # nvidia-settings : NVIDIA X11 settings GUI
  # ocl-icd : OpenCL ICD loader
  # opencl-nvidia : NVIDIA OpenCL implementation
  # lib32-opencl-nvidia : 32-bit OpenCL (for Proton)
  # clinfo : query OpenCL platforms/devices
  # cuda : CUDA toolkit/runtime

  yay -S --needed \
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

Extra stuffs for gaming (NVIDIA only):

```bash
if [ "$gpu_vendor" = "nvidia" ]; then
  yay -S --needed \
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

Some launch option for Steam/Proton (use in Steam launch option, NVIDIA only):

```bash
# PROTON_ENABLE_NVAPI=1 DXVK_ASYNC=1 %command%
```

Set Shader Cache Size (NVIDIA only):

```bash
if [ "$gpu_vendor" = "nvidia" ]; then
  mkdir -p ~/.nv
  echo "GLShaderDiskCacheSize=17179869184" > ~/.nv/nvidia-application-profiles-rc
fi
```

<details>
  <summary>🟩 Packages being installed (NVIDIA Open DKMS + CUDA/OpenCL)</summary>

- Driver (DKMS): `nvidia-open-dkms`, `nvidia-utils`, `lib32-nvidia-utils`, `nvidia-settings`
- OpenCL: `ocl-icd`, `opencl-nvidia`, `lib32-opencl-nvidia`, `clinfo`
- CUDA toolkit/runtime: `cuda`

</details>

#### Add DRM kernel module (NVIDIA only)

Only run this section if an NVIDIA GPU was detected.

You should have `/boot/limine/limine.conf` created automatically by now. Edit it and add the NVIDIA params to the `CMDLINE:` lines. You can leave out the fallback entry, or keep it as a safe mode option.

```bash
# Only if GPU is NVIDIA
nvidia-drm.modeset=1 nvidia-drm.fbdev=1
```

In `/etc/mkinitcpio.conf`, ensure NVIDIA modules are included and the generic `kms` hook is removed:

```bash
if [ "$gpu_vendor" = "nvidia" ]; then
  sudo perl -0777 -i.bak -pe '
    # Ensure NVIDIA modules exist in MODULES=()
    s{^(?!\\s*#)\\s*MODULES=\\(([^)]*)\\)\\s*$}{
      my @mods = grep { length } split " ", $1;          # split on whitespace
      my %seen; @mods = grep { !$seen{$_}++ } @mods;     # de-dup, keep order
      for my $add (qw(nvidia nvidia_modeset nvidia_uvm nvidia_drm)) {
        push @mods, $add unless $seen{$add}++;
      }
      "MODULES=(" . join(" ", @mods) . ")"
    }mge;

    # Remove kms from HOOKS=() (if present)
    s{^(?!\\s*#)\\s*HOOKS=\\(([^)]*)\\)\\s*$}{
      my @hooks = grep { length && $_ ne "kms" } split " ", $1;
      "HOOKS=(" . join(" ", @hooks) . ")"
    }mge;
  ' /etc/mkinitcpio.conf

  # Rebuild initramfs
  sudo mkinitcpio -P

  # Verify
  grep -E '^(MODULES|HOOKS)=' /etc/mkinitcpio.conf
fi
```

Add a NVIDIA pacman hook so DKMS/initramfs are rebuilt automatically when kernels or the driver are updated:

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
Target = linux-cachyos
Target = linux-cachyos-eevdf
Target = linux

# Adjust line(6) above to match your driver, e.g. Target=nvidia-dkms for proprietary or Target=nvidia-470xx-dkms for legacy GPUs

# Change line(7) above, if you are not using CachyOS kernels. For example, Target=linux-lts

[Action]
Description = Update Nvidia module in initramfs
Depends = mkinitcpio
When = PostTransaction
NeedsTargets
Exec = /bin/sh -c 'while read -r trg; do case $trg in linux) exit 0; esac; done; /usr/bin/mkinitcpio -P'
EOF
fi
```

Then trigger Limine sync (if NVIDIA was installed) and reboot:

```bash
if [ "$gpu_vendor" = "nvidia" ]; then
  # Disable old limine.conf since limine-snapper-sync generates a new one
  sudo mv /boot/limine/limine.conf /boot/limine/limine.conf.bak 2>/dev/null || true
fi

reboot
```

### Disable KWallet prompts (optional)

This disables the KDE Secret Service backend; apps like VS Code, Git credential helpers, and browsers won’t retain secrets. Skip this if you followed the KWallet setup above.

```bash
mkdir -p ~/.config
cat <<'EOF' >> ~/.config/kwalletrc
[Wallet]
Enabled=false
EOF

```

### Clear package manager caches

```bash
yay -Syu
sudo paccache -r
yay -Yc

```

## pyenv

🐍 Goal: install `pyenv` so you can manage multiple Python versions per‑user, and wire it into your shell.

### Install build dependencies and pyenv

```bash

# Install common build deps for compiling Python versions

sudo pacman -S --needed base-devel openssl zlib xz tk readline sqlite libffi bzip2

# Clone pyenv into your home directory

git clone https://github.com/pyenv/pyenv.git ~/.pyenv

```

### Shell integration (bash/zsh)

```bash
shell_name="$(basename "${SHELL:-bash}")"
conf="$HOME/.bashrc"
if [ "$shell_name" = "zsh" ]; then conf="$HOME/.zshrc"; fi
touch "$conf"

# Add pyenv init block (idempotent)

if ! grep -qF 'PYENV_ROOT' "$conf"; then
  cat <<'EOF' >> "$conf"

# pyenv

export PYENV_ROOT="$HOME/.pyenv"
export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init -)"
EOF
fi

```

### Use pyenv to install a Python version

```bash

# List available Python versions

pyenv install --list | grep " 3\.13"

# Install a specific version and make it the default for your user

# Arch system Python is 3.13+ as of 2025. Use pyenv for your preferred version.

pyenv install 3.13.2
pyenv global 3.13.2

# Confirm which Python is active

python --version
pyenv which python

```

## Flatpak Apps

Can’t do this over SSH; install this in the logged-in session.

```bash
flatpak install -y flathub \
  org.gnome.baobab \
  io.github.jonmagon.kdiskmark \
  io.github.flattool.Warehouse \
  io.missioncenter.MissionCenter \
  xyz.z3ntu.razergenie \
  io.github.arunsivaramanneo.GPUViewer \
  org.kde.kdf \
  com.kgurgul.cpuinfo \
  org.raspberrypi.rpi-imager \
  io.github.shonubot.Spruce \
  org.localsend.localsend_app \
  com.agateau.nanonote \
  org.gnome.Cheese \
  com.discordapp.Discord \
  com.cassidyjames.butler \
  com.adobe.Reader \
  net.davidotek.pupgui2 \
  com.vysp3r.ProtonPlus

```

## Theme

🎨 These are my personal favorites — use them as a starting point and change anything you don’t like.

Stutter note: Whitesur, Orchis, and Colloid themes caused window-resize stutter on my system. Qogir, Darkly, and Vinyl didn’t. Hardware may change results, so try a few before settling.

KDE settings path: **System Settings → Appearance**.

> **Note:** SDDM theme install scripts below work for SDDM. If using `plasma-login-manager` (the Plasma 6.6+ default), skip the SDDM theme installs -- Plasma Login Manager uses Plasma's own theming system via System Settings -> Login Screen.

### DE Themes

```bash

# Whitesur KDE

repo=~/Desktop/theme
git clone https://github.com/vinceliuice/WhiteSur-kde.git $repo
cd $repo
./install.sh
cd sddm && sudo ./install.sh  # sddm
rm -rf $repo && cd ~

# Whitesur GTK

repo=~/Desktop/theme
git clone https://github.com/vinceliuice/WhiteSur-gtk-theme.git $repo
cd $repo
./install.sh
sudo flatpak override --filesystem=xdg-config/gtk-3.0 && sudo flatpak override --filesystem=xdg-config/gtk-4.0
rm -rf $repo && cd ~

# Mojave GTK

repo=~/Desktop/theme
git clone https://github.com/vinceliuice/Mojave-gtk-theme.git $repo
cd $repo
./install.sh -l -g
rm -rf $repo && cd ~

# Orchis KDE

repo=~/Desktop/theme
git clone https://github.com/vinceliuice/Orchis-kde.git $repo
cd $repo
./install.sh
cd sddm && sudo ./install.sh  # sddm
rm -rf $repo && cd ~

# Orchis GTK

repo=~/Desktop/theme
git clone https://github.com/vinceliuice/Orchis-theme.git $repo
cd $repo
./install.sh
rm -rf $repo && cd ~

# Colloid KDE

repo=~/Desktop/theme
git clone https://github.com/vinceliuice/Colloid-kde.git $repo
cd $repo
./install.sh
cd sddm/6.0 && sudo ./install.sh
rm -rf $repo && cd ~

# Qogir KDE

repo=~/Desktop/theme
git clone https://github.com/vinceliuice/Qogir-kde.git $repo
cd $repo
./install.sh
cd sddm && sudo ./install.sh  # sddm
rm -rf $repo && cd ~

# MacTahoe KDE

repo=~/Desktop/theme
git clone https://github.com/vinceliuice/MacTahoe-kde.git $repo
cd $repo
./install.sh
cd sddm && sudo ./install.sh  # sddm
rm -rf $repo && cd ~

# Vinyl

yay -Syu
yay -S --needed base-devel vinyl

```

### Icon themes & cursors

```bash

# Tela-circle-icon-theme

repo=~/Desktop/icons
git clone https://github.com/vinceliuice/Tela-circle-icon-theme.git $repo
cd $repo
./install.sh standard black blue nord
rm -rf $repo && cd ~

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
cd cursor && sudo ./install.sh    # cursors
rm -rf $repo && cd ~

```

## Credits & Thanks

🙏 Huge thanks to the authors/maintainers of these guides and notes that helped shape parts of this memo:

- `fstab` notes: <https://gist.github.com/mjkstra/96ce7a5689d753e7a6bdd92cdc169bae#fstab>
- Arch install walkthrough/reference: <https://github.com/silentz/arch-linux-install-guide>
- Chroot exit + reboot checklist: <https://gist.github.com/yovko/512326b904d120f3280c163abfbcb787#exit-from-chroot-and-reboot>
- NVIDIA driver guide: <https://github.com/korvahannu/arch-nvidia-drivers-installation-guide>
