# Arch Linux + Btrfs + linux-zen + GRUB + KDE Plasma + Nvidia Driver

Personal notes for rebuilding my daily Arch install: UEFI firmware, single NVMe drive, Btrfs root, KDE Plasma desktop, GRUB bootloader, and ready for dual-booting, Nvidia DKMS, and Snapper-style snapshots.

Not the best way or most correct way. Just the way I like.

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
12. [Post-Install Ideas](#post-install-ideas)

## Updates

### 2026-03-21

- Creating this GRUB-based variant because Limine cannot be installed on a Btrfs `/boot` subvolume. I want `/boot` on Btrfs so it can be included in snapshots.
- This guide drops CachyOS kernels and uses `linux-zen` instead.

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

**CAUTION:** double-check your device paths (e.g. `/dev/nvme0n1p3`) before formatting.

```bash

# Change the disk path to match yours!!!

mkfs.btrfs -f -L ArchLinuxFS /dev/nvme0n1p3
mkswap /dev/nvme0n1p2

# Store UUID for later scripts.

esp_uuid="$(blkid -s UUID -o value /dev/nvme0n1p1)"
swap_uuid="$(blkid -s UUID -o value /dev/nvme0n1p2)"
root_uuid="$(blkid -s UUID -o value /dev/nvme0n1p3)"

```

## Btrfs Subvolumes & Mounts

🧱 Goal: create a subvolume layout that plays nicely with snapshot tools (e.g., Snapper) and keeps "churny" paths isolated (logs/cache). This is a personal layout; feel free to add/remove subvolumes depending on what you plan to snapshot.

📝 Notes:

- Subvolumes here are named `@something` by convention; `@` is mounted as `/` later.
- `compress=zstd:1,noatime` is a common baseline; tune for your workload.
- If you don't care about isolating `/var/log` or `/var/cache`, you can skip those subvolumes and mounts.
- I have Downloads, `.cache`, and Git isolated. These usually get big and I don't want to snapshot them.

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
btrfs subvolume create /mnt/@boot

# OPTIONAL: extra subvolumes

# I find these folders usually grow large on my system and they change quite often, so I opted not to snapshot them.

# You can skip these. Especially `home/Git` — that's my place for all git repos.

# remount as btrfs subvolumes

umount -R /mnt
mount -o compress=zstd:1,noatime,subvol=@ UUID="${root_uuid}" /mnt

mount --mkdir -o compress=zstd:1,noatime,subvol=@home UUID="${root_uuid}" /mnt/home
mount --mkdir -o compress=zstd:1,noatime,subvol=@var  UUID="${root_uuid}" /mnt/var
mount --mkdir -o compress=zstd:1,noatime,subvol=@var_log  UUID="${root_uuid}" /mnt/var/log
mount --mkdir -o compress=zstd:1,noatime,subvol=@var_cache UUID="${root_uuid}" /mnt/var/cache
mount --mkdir -o compress=zstd:1,noatime,subvol=@root UUID="${root_uuid}" /mnt/root
mount --mkdir -o compress=zstd:1,noatime,subvol=@boot UUID="${root_uuid}" /mnt/boot

# If you created optional subvolumes, mount them.

```

### 3.2 Swap, ESP and fstab

📝 **Note:** The ESP (FAT32) is mounted at `/boot/EFI`, while `/boot` itself resides on the Btrfs subvolume. This keeps kernel/initramfs files snapshot-able on Btrfs while GRUB writes to the FAT32 ESP at `/boot/EFI`.

```bash

# Mount ESP at /boot/EFI (Btrfs /boot subvolume is already mounted)

mount --mkdir UUID="${esp_uuid}" /mnt/boot/EFI

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
  linux linux-headers linux-zen linux-zen-headers linux-firmware "${cpu}-ucode" \
  efibootmgr btrfs-progs dosfstools e2fsprogs exfatprogs\
  networkmanager openssh \
  vim nvim git sudo man curl wget perl \
  zsh zsh-completions zsh-autosuggestions bash-completion \
  pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber reflector \
  inotify-tools

# copy pacman config

cp /etc/pacman.conf /mnt/etc/pacman.conf
cp /etc/pacman.d/mirrorlist /mnt/etc/pacman.d/mirrorlist

```

<details>
  <summary>📦 Packages being installed (Base Install)</summary>

- Core system: `base`, `base-devel`
- Kernel + firmware: `linux-zen`, `linux-zen-headers`, `linux-firmware`
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

📝 **Note:** Pick a hostname that identifies this machine (e.g., `arch`, `desktop`, `workstation`, or any name you prefer). Keep it short and lowercase.

```bash
host_name="arch"
```

```bash
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

📝 **Note:** Change `user=pop` to your desired username.

```bash
user=pop
```

```bash
echo "Create user, and set password"
useradd -m -G wheel,storage,power,audio,video -s /bin/bash $user
passwd $user
```

**Enable wheel sudo in `sudoers`**
We won't automate this.

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

## GRUB Bootloader

🚀 Goal: install GRUB to the EFI System Partition and generate a `grub.cfg` that boots your Arch kernel/initramfs from Btrfs by UUID. GRUB will be installed to `/boot/EFI` (the FAT32 ESP), while kernel and initramfs files live on the Btrfs `/boot` subvolume.

### 6.1 Clean up EFI boot entries (optional)

📝 **Note:** On a fresh install or if you no longer need other boot entries (e.g., Windows Boot Manager), you can remove old EFI boot entries to clean up your UEFI boot menu. **Do not do this if you want to keep other OS boot options.**

```bash

# View current EFI boot entries

efibootmgr -v

# Delete unwanted boot entries (replace XXXX with the boot number, e.g., 0001, 0002)

efibootmgr --delete-bootnum --bootnum XXXX

```

📝 **Tip:** Keep only the GRUB entry. You can identify entries by their label (e.g., "Windows Boot Manager", "GRUB").

### 6.2 Install GRUB packages

```bash

# GRUB and required tools

pacman -S --needed grub efibootmgr dosfstools

# Install GRUB to the EFI System Partition

grub-install --target=x86_64-efi --efi-directory=/boot/EFI --bootloader-id=GRUB

```

### 6.3 Configure GRUB cmdline

📝 **Note:** Set up kernel parameters in `/etc/default/grub` for Btrfs root, resume (hibernation), and optional zswap support.

```bash

# Detect CPU and get UUIDs

ucode_img="intel"
lscpu | grep -qi amd && ucode_img="amd"
swap_uuid="$(blkid -s UUID -o value /dev/nvme0n1p2)"

# Remove any existing GRUB_CMDLINE_LINUX_DEFAULT line

sed -i '/^GRUB_CMDLINE_LINUX_DEFAULT=/d' /etc/default/grub

# Add the new line (shell expands variables before writing)

echo "GRUB_CMDLINE_LINUX_DEFAULT=\"loglevel=3 resume=UUID=${swap_uuid} zswap.enabled=1 zswap.compressor=lz4 zswap.max_pool_percent=50 zswap.zpool=zsmalloc ${ucode_img}_iommu=on iommu=pt\"" >> /etc/default/grub

# Verify the change

grep "^GRUB_CMDLINE_LINUX_DEFAULT=" /etc/default/grub

```

📝 **Tip:** If you don't want zswap, remove the `zswap.*` parameters from the cmdline.

### 6.3.1 Add zswap modules to initramfs (optional)

```bash

# Add zswap modules to mkinitcpio.conf (idempotent)

perl -0777 -i.bak -pe '
  s{^(?!\s*#)\s*MODULES=\(([^)]*)\)\s*$}{
    my @mods = grep { length } split " ", $1;
    my %seen; @mods = grep { !$seen{$_}++ } @mods;
    for my $add (qw(lz4 lz4_compress zsmalloc)) {
      push @mods, $add unless $seen{$add}++;
    }
    "MODULES=(" . join(" ", @mods) . ")"
  }mge;
' /etc/mkinitcpio.conf

# Rebuild initramfs

mkinitcpio -P

# Verify

grep -E "^MODULES=" /etc/mkinitcpio.conf

```

### 6.4 Generate GRUB configuration

```bash

# Generate grub.cfg

grub-mkconfig -o /boot/grub/grub.cfg

```

### 6.4 Enable os-prober (optional, for dual-boot)

📝 **Note:** If you have Windows or another Linux installation on the same machine, enable `os-prober` to detect and add them to the GRUB menu.

```bash

# Install os-prober

pacman -S --needed os-prober

# Enable os-prober in GRUB config

echo 'GRUB_DISABLE_OS_PROBER=false' >> /etc/default/grub

# Regenerate GRUB config

grub-mkconfig -o /boot/grub/grub.cfg

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

🖥️ Goal: install KDE Plasma + a display manager (SDDM) and the integration pieces you'll want on a typical desktop (portals, NetworkManager applet, audio, thumbnails). If you don't want many of the KDE apps, you can skip the Desktop Apps section.

### KDE Plasma & apps

#### KDE Core

```bash
# sddm : Display/login manager (graphical login screen)
# sddm-kcm : System Settings module to configure SDDM
# xdg-desktop-portal : "Portal" framework (file picker, screen share, sandbox app integration)
# xdg-desktop-portal-kde : KDE backend for portals (needed for Wayland screen share, Flatpak, etc.)
# qt6-wayland : Qt6 Wayland platform plugin
# xorg-xwayland : Runs X11 apps under Wayland

pacman -S --needed \
  sddm \
  sddm-kcm \
  qt5-declarative \
  xdg-desktop-portal \
  xdg-desktop-portal-kde \
  qt6-wayland \
  xorg-xwayland

systemctl enable sddm
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

# Optional: Encrypted "vault" folders integration: plasma-vault
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

# Add PAM hooks for SDDM so the wallet unlocks with your session.
# If you use a different display manager, add pam_kwallet5 there instead.

if ! grep -q pam_kwallet5 /etc/pam.d/sddm; then
  sudo tee -a /etc/pam.d/sddm >/dev/null <<'EOF'
auth       optional pam_kwallet5.so
session    optional pam_kwallet5.so auto_start
EOF
fi
```

- After logging in, open System Settings → KDE Wallet and ensure a wallet exists (Blowfish is fine). This keeps VS Code and other apps from nagging when storing secrets.

#### Desktop Apps

```bash
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
# btop : System monitor (alternative to htop)
# fastfetch : System info tool (neofetch alternative)

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
  kcalc \
  btop \
  fastfetch
```

<details>
  <summary>🖥️ Packages being installed (KDE Plasma desktop)</summary>

- Display manager: `sddm`, `sddm-kcm`
- Portals/Wayland helpers: `xdg-desktop-portal`, `xdg-desktop-portal-kde`, `qt6-wayland`, `xorg-xwayland`
- Plasma core: `plasma-desktop`, `plasma-workspace`, `plasma-wayland-session`, `kwin`, `systemsettings`
- Plasma integration: `plasma-nm`, `plasma-pa`, `kscreen`, `kde-gtk-config`, `breeze-gtk`
- Optional Plasma extras: `bluedevil`, `power-profiles-daemon`, `kdeplasma-addons`, `plasma-systemmonitor`, `plasma-browser-integration`, `discover`, `krdp`, `print-manager`
- Desktop apps: `dolphin`, `dolphin-plugins`, `konsole`, `kate`, `okular`, `gwenview`, `spectacle`, `ark`, `gparted`, `filelight`, `kcalc`
- Thumbnails/integration: `kio-extras`, `ffmpegthumbs`, `kdegraphics-thumbnailers`

</details>

## Reboot

🔁 Goal: cleanly exit the installer environment and reboot into the new system. Before rebooting, it's worth quickly checking `/mnt/etc/fstab` and that GRUB is properly installed.

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

## Post-Install

Log in to your new system using your user account.

### Create XDG user directories

```bash

# Create standard user folders (Desktop, Documents, Downloads, etc.)

xdg-user-dirs-update

```

## SSH Setup

🔐 Goal: set up SSH key-based authentication for secure remote access to your new system. We were using password login for the live ISO; now let's configure proper SSH keys.

### Generate SSH key pair (on your client machine)

```bash

# On your client machine (not the Arch system), generate an Ed25519 key

ssh-keygen -t ed25519 -C "your_email@example.com"

# Or use RSA 4096-bit if Ed25519 is not supported

# ssh-keygen -t rsa -b 4096 -C "your_email@example.com"

```

### Copy SSH key to Arch system

```bash

# From your client machine, copy the public key to your Arch system
# Replace 'pop' with your username and 'arch' with your hostname/IP

ssh-copy-id -i ~/.ssh/id_ed25519.pub pop@arch

# Or manually copy if ssh-copy-id is not available

cat ~/.ssh/id_ed25519.pub | ssh pop@arch "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"

```

### SSH client config (on your client machine)

```bash

# Edit ~/.ssh/config on your client machine

cat >> ~/.ssh/config <<'EOF'

# Arch Linux desktop

Host arch
    HostName 192.168.1.100
    User pop
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
    ForwardAgent yes
    ForwardX11 yes
EOF

```

📝 **Note:** Change `HostName` to your Arch system's actual IP address.

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

# Disable challenge-response authentication
ChallengeResponseAuthentication no

# Use SSH protocol 2 only
Protocol 2

# Limit max authentication attempts
MaxAuthTries 3

# Set login grace time (60 seconds)
LoginGraceTime 60

# Disable X11 forwarding if not needed (optional)
# X11Forwarding no

# Enable strict mode
StrictModes yes

# Disable host-based authentication
HostbasedAuthentication no
IgnoreRhosts yes

# Set client alive interval for idle timeout (optional)
ClientAliveInterval 300
ClientAliveCountMax 2
EOF

```

#### Optional: Change SSH port (security through obscurity)

```bash

# Change SSH port from default 22 to a custom port (e.g., 2222)

sudo sed -i 's/^#Port 22/Port 2222/' /etc/ssh/sshd_config

# Update firewall if using one

# sudo ufw allow 2222/tcp

```

📝 **Note:** Remember to specify the port when connecting: `ssh -p 2222 arch`

#### Optional: Allow only specific users

```bash

# Add to /etc/ssh/sshd_config

echo "AllowUsers pop" | sudo tee -a /etc/ssh/sshd_config

```

#### Restart SSH service

```bash

# After making changes, restart SSHD

sudo systemctl restart sshd

# Verify SSH is running

sudo systemctl status sshd

```

#### Test before disconnecting!

⚠️ **Important:** Before closing your current SSH session, open a new terminal and test that you can connect with your key:

```bash
ssh arch

```

If it works, you're all set. If not, you still have the active session to fix any issues.

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

📸 Goal: set up snapshot management for Btrfs so you can roll back system changes.

```bash
yay -Suy --needed snapper grub-btrfs btrfs-assistant snap-pac snap-pac-grub snapper-gui-git snapper-tools

```

<details>
  <summary>📸 Packages being installed (Snapper + GRUB integration)</summary>

- `snapper`: manage Btrfs snapshots (create/list/rollback policies)
- `grub-btrfs`: GRUB menu entries for Btrfs snapshots
- `btrfs-assistant`: GUI for Btrfs management
- `snap-pac`: pacman hooks to auto-create snapshots around package transactions
- `snap-pac-grub`: update GRUB after snapshot operations
- `snapper-gui-git`: GUI for snapper snapshot management (AUR)
- `snapper-tools`: additional snapper utilities (AUR)

</details>

#### Enable grub-btrfs service

```bash

# grub-btrfs will auto-generate GRUB entries for snapshots

sudo systemctl enable --now grub-btrfsd

```

#### Create Snapper Configs

You can omit `/home` snapping if you prefer (e.g., for large media folders).

```bash

# We need to be root for this part

sudo su

```

```bash
snapper -c root create-config /
snapper -c boot create-config /boot
snapper -c home create-config /home

```

```bash

# Exit root

exit

```

### Extra Packages and fonts

🧺 Personal "daily driver" pick list. Install only what you want.

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

#### JavaScript/TypeScript runtimes

```bash
yay -S --needed nodejs npm bun
```

#### Desktop apps

```bash
yay -S --needed --noconfirm\
  firefox chromium brave-bin \
  libreoffice-fresh \
  mailspring-bin \
  visual-studio-code-bin \
  filezilla \
  fatfetch

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
yay -S --needed --noconfirm\
  noto-fonts \
  noto-fonts-emoji \
  ttf-dejavu \
  ttf-ubuntu-font-family \
  terminus-font \
  nerd-fonts \
  ttf-ms-fonts

```

### SPDIF audio dropout / sleep

🔊 Some SPDIF receivers/DACs go to sleep after a few idle seconds, so the first 1-3 seconds of the next sound get eaten while the link wakes up. Two tweaks that keep the SPDIF clock alive:

1) Disable codec autosuspend (system-wide)

```bash
sudo tee /etc/modprobe.d/alsa-no-powersave.conf >/dev/null <<'EOF'
options snd_hda_intel power_save=0 power_save_controller=N
EOF

```

Reboot for this to take effect.

2) Stop WirePlumber from suspending the sink (per-user)

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

### Nvidia Driver

🟩 This is a rough checklist for an NVIDIA DKMS setup. Exact package names and kernel module steps depend on your GPU generation and kernel choice, so verify against the Arch Wiki for your hardware: <https://wiki.archlinux.org/title/NVIDIA>

Pick one path below (match to your GPU/needs).

#### Option A: `nvidia-open` (590xx, repo driver)

> **Note:** The current Nvidia driver (590xx series) does not have a proprietary `nvidia-dkms` package in the official repositories, so `nvidia-open-dkms` is used instead. The downside is that GSP firmware cannot be disabled in the open driver (see Arch Wiki).

```bash

# nvidia-open-dkms : NVIDIA DKMS driver (kernel modules)
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

```

#### Option B: Proprietary 580xx (AUR)

Use this if you want the proprietary stack or run into issues with the 590xx open driver.

```bash
yay -S --needed \
  nvidia-580xx-dkms \
  nvidia-580xx-utils \
  lib32-nvidia-580xx-utils \
  nvidia-580xx-settings \
  opencl-nvidia-580xx \
  lib32-opencl-nvidia-580xx \
  libxnvctrl-580xx \
  ocl-icd \
  clinfo \
  cuda

```

Extra stuffs for gaming

```bash
yay -S --needed \
  libva-utils \
  vdpauinfo \
  vulkan-tools \
  libva-nvidia-driver \
  dxvk \
  vkd3d \
  shaderc \
  spirv-tools

```

Some launch option for Steam/Proton (use in Steam launch option)

```bash
  PROTON_ENABLE_NVAPI=1 DXVK_ASYNC=1 %command%

```

Set Shader Cache Size

```bash
mkdir ~/.nv
echo "GLShaderDiskCacheSize=17179869184" > ~/.nv/nvidia-application-profiles-rc

```

<details>
  <summary>🟩 Packages being installed (NVIDIA DKMS + CUDA/OpenCL)</summary>

- Driver (DKMS): `nvidia-open-dkms`, `nvidia-utils`, `lib32-nvidia-utils`, `nvidia-settings` (swap to `nvidia-580xx-*` for Option B)
- OpenCL: `ocl-icd`, `opencl-nvidia`, `lib32-opencl-nvidia`, `clinfo` (use `*-580xx` variants for Option B)
- CUDA toolkit/runtime: `cuda` (works with both options)

</details>

#### Add DRM kernel module

Edit `/etc/default/grub` and add the NVIDIA params to `GRUB_CMDLINE_LINUX_DEFAULT`:

```bash
nvidia-drm.modeset=1 nvidia-drm.fbdev=1

```

In `/etc/mkinitcpio.conf`, ensure NVIDIA modules are included and the generic `kms` hook is removed:

```bash

sudo perl -0777 -i.bak -pe '
  # Ensure NVIDIA modules exist in MODULES=()
  s{^(?!\s*#)\s*MODULES=\(([^)]*)\)\s*$}{
    my @mods = grep { length } split " ", $1;          # split on whitespace
    my %seen; @mods = grep { !$seen{$_}++ } @mods;     # de-dup, keep order
    for my $add (qw(nvidia nvidia_modeset nvidia_uvm nvidia_drm)) {
      push @mods, $add unless $seen{$add}++;
    }
    "MODULES=(" . join(" ", @mods) . ")"
  }mge;

  # Remove kms from HOOKS=() (if present)
  s{^(?!\s*#)\s*HOOKS=\(([^)]*)\)\s*$}{
    my @hooks = grep { length && $_ ne "kms" } split " ", $1;
    "HOOKS=(" . join(" ", @hooks) . ")"
  }mge;
' /etc/mkinitcpio.conf

# Rebuild initramfs

sudo mkinitcpio -P

# Verify

grep -E '^(MODULES|HOOKS)=' /etc/mkinitcpio.conf

```

Add a NVIDIA pacman hook so DKMS/initramfs are rebuilt automatically when kernels or the driver are updated:

```bash

# ensure hooks directory exists

sudo mkdir -p /etc/pacman.d/hooks/

# hook: rebuild NVIDIA initramfs on updates

sudo tee /etc/pacman.d/hooks/nvidia.hook >/dev/null <<'EOF'
[Trigger]
Operation = Install
Operation = Upgrade
Operation = Remove
Type = Package
Target = nvidia-open-dkms
Target = linux
Target = linux-zen

# Adjust line(6) above to match your driver, e.g. Target=nvidia-580xx-dkms (Option B) or Target=nvidia-470xx-dkms

# Change line(7) above, if you are not using the regular kernel For example, Target=linux-lts

[Action]
Description = Update Nvidia module in initcpio
Depends = mkinitcpio
When = PostTransaction
NeedsTargets
Exec = /bin/sh -c 'while read -r trg; do case $trg in linux) exit 0; esac; done; /usr/bin/mkinitcpio -P'
EOF

```

Then regenerate GRUB config and reboot:

```bash
grub-mkconfig -o /boot/grub/grub.cfg
reboot

```

### Disable KWallet prompts (optional)

This disables the KDE Secret Service backend; apps like VS Code, Git credential helpers, and browsers won't retain secrets. Skip this if you followed the KWallet setup above.

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

pyenv install --list | grep " 3\.12"

# Install a specific version and make it the default for your user

pyenv install 3.12.2
pyenv global 3.12.2

# Confirm which Python is active

python --version
pyenv which python

```

## Flatpak Apps

Can't do this over SSH; install this in the logged-in session.

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

🎨 These are my personal favorites — use them as a starting point and change anything you don't like.

Stutter note: Whitesur, Orchis, and Colloid themes caused window-resize stutter on my system. Qogir, Darkly, and Vinyl didn't. Hardware may change results, so try a few before settling.

KDE settings path: **System Settings → Appearance**.

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
cd cursor && sudo ./install.sh    # cursors
rm -rf $repo && cd ~

```

## Credits & Thanks

🙏 Huge thanks to the authors/maintainers of these guides and notes that helped shape parts of this memo:

- `fstab` notes: <https://gist.github.com/mjkstra/96ce7a5689d753e7a6bdd92cdc169bae#fstab>
- Arch install walkthrough/reference: <https://github.com/silentz/arch-linux-install-guide>
- Chroot exit + reboot checklist: <https://gist.github.com/yovko/512326b904d120f3280c163abfbcb787#exit-from-chroot-and-reboot>
- NVIDIA driver guide: <https://github.com/korvahannu/arch-nvidia-drivers-installation-guide>
