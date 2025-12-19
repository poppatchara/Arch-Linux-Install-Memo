# Arch Linux + Btrfs + CachyOS Kernels + KDE Plasma + Limine + Nvidia Driver

Personal notes for rebuilding my daily Arch install: UEFI firmware, single NVMe drive, Btrfs root, KDE Plasma desktop, Limine bootloader, and ready for dual-booting, Nvidia DKMS, and Snapper-style snapshots.

## Contents
1. [Assumptions](#assumptions)
2. [Live ISO Prep](#live-iso-prep)
3. [Partition & Format](#partition--format)
4. [Btrfs Subvolumes & Mounts](#btrfs-subvolumes--mounts)
5. [Base Install](#base-install)
6. [Chroot Configuration](#chroot-configuration)
7. [Limine Bootloader](#limine-bootloader)
8. [Services & QoL](#services--qol)
9. [Desktop Stack](#desktop-stack)
10. [Snapper](#snapper)
11. [Trim & Reboot](#trim--reboot)
12. [Post-Install Ideas](#post-install-ideas)
13. [Credits & Thanks](#credits--thanks)

---

## Assumptions
🧩 This memo is written as a linear “do this, then that” install. If you deviate (different disk name, multiple disks, LUKS, existing Windows ESP, etc.), adjust the affected commands and double-check mounts/UUIDs before continuing.
- Arch ISO already written to USB (any recent release).
- I SSH into the live ISO from another machine (root password set with `passwd`, IP from `ip a`), purely for copy/paste convenience.
- Secure Boot disabled, UEFI mode enabled.
- Target disk: `/dev/nvme0n1` (adjust commands if yours differs).
- Wired Ethernet or reliable Wi-Fi (`iwctl`) available.
- Timezone `Asia/Bangkok`; hostname defaults to `arch`.
- EFI System Partition (ESP) already exists as the first partition; swap is the last partition to simplify potential resizing.

---

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
Reorder the existing mirrorlist: move your `## <country>` section to right before `## Worldwide`, with exactly one blank line between them.
```bash
# Reorder the mirrorlist so the `## $country` block sits right before `## Worldwide`.
# This keeps your local mirrors near the top without touching the rest of the file.
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

---

## Partition & Format
💽 Goal: create a minimal GPT layout (ESP + Btrfs root + swap) and capture stable UUIDs for later bootloader and `fstab` configuration. Before running format commands, sanity-check your target disk with `lsblk -f` so you don’t wipe the wrong device.

### 1. Partition layout
| Partition | Size | Type | Purpose |
|-----------|------|------|---------|
| `/dev/nvme0n1p1` | 4 GiB | EFI System (type `ef00`) | Mounted at `/boot` |
| `/dev/nvme0n1p2` | Remainder | Linux filesystem (`8300`) | Btrfs root |
| `/dev/nvme0n1p3` | ≈ RAM size | Linux swap | Swap |

Use `cfdisk /dev/nvme0n1` (GPT) to build or adjust the layout.

### 2. Format and capture UUIDs
**CAUTION:** double-check your device paths (e.g. `/dev/nvme0n1p2`) before formatting.
```bash
mkfs.btrfs -f -L ArchLinuxFS /dev/nvme0n1p2
mkswap /dev/nvme0n1p3

esp_uuid="$(blkid -s UUID -o value /dev/nvme0n1p1)"
root_uuid="$(blkid -s UUID -o value /dev/nvme0n1p2)"
swap_uuid="$(blkid -s UUID -o value /dev/nvme0n1p3)"
```

---

## Btrfs Subvolumes & Mounts
🧱 Goal: create a subvolume layout that plays nicely with snapshot tools (e.g., Snapper) and keeps “churny” paths isolated (logs/cache). This is a personal layout; feel free to add/remove subvolumes depending on what you plan to snapshot.

📝 Notes:
- Subvolumes here are named `@something` by convention; `@` is mounted as `/` later.
- `compress=zstd:1,noatime` is a common baseline; tune for your workload.
- If you don’t care about isolating `/var/log` or `/var/cache`, you can skip those subvolumes and mounts.

```bash
# Create subvolumes
mount UUID="${root_uuid}" /mnt
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@var
btrfs subvolume create /mnt/@log
btrfs subvolume create /mnt/@cache
btrfs subvolume create /mnt/@root
btrfs subvolume create /mnt/@srv
umount /mnt

# Mount subvolumes
mount -o compress=zstd:1,noatime,subvol=@ UUID="${root_uuid}" /mnt
mount --mkdir -o compress=zstd:1,noatime,subvol=@home UUID="${root_uuid}" /mnt/home
mount --mkdir -o compress=zstd:1,noatime,subvol=@var  UUID="${root_uuid}" /mnt/var
mount --mkdir -o compress=zstd:1,noatime,subvol=@log  UUID="${root_uuid}" /mnt/var/log
mount --mkdir -o compress=zstd:1,noatime,subvol=@cache UUID="${root_uuid}" /mnt/var/cache
mount --mkdir -o compress=zstd:1,noatime,subvol=@root UUID="${root_uuid}" /mnt/root
mount --mkdir -o compress=zstd:1,noatime,subvol=@srv  UUID="${root_uuid}" /mnt/srv

# Mount ESP at /boot
mount --mkdir UUID="${esp_uuid}" /mnt/boot

# Enable swap
swapon UUID="${swap_uuid}"

# Generate fstab
mkdir -p /mnt/etc
genfstab -U /mnt > /mnt/etc/fstab
cat /mnt/etc/fstab
```

---

## Base Install
📦 Goal: install a bootable base system onto `/mnt` (kernel, firmware, networking, editors, etc.), then switch into it with `arch-chroot`. If you prefer fewer packages, trim this list, but keep `base`, a kernel, `linux-firmware`, and whatever you need for networking and your filesystem.

```bash
# Detect arch
ucode_pkg=intel-ucode
if lscpu | grep -qi amd; then
  ucode_pkg=amd-ucode
fi

# Prepare vconsole
cat <<'EOF' > /mnt/etc/vconsole.conf
KEYMAP=us
FONT=
EOF

# Installations
pacstrap -K /mnt \
  base base-devel \
  linux linux-headers linux-firmware "${ucode_pkg}" \
  efibootmgr btrfs-progs  dosfstools \
  networkmanager openssh \
  vim nvim git sudo man curl wget perl \
  zsh zsh-completions zsh-autosuggestions bash-completion \
  pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber reflector \
  inotify-tools 

arch-chroot /mnt
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

Wait for the install to finish, then continue inside the chroot.

## Chroot Configuration
🏠 Everything below runs inside `arch-chroot /mnt` unless explicitly stated otherwise. This section sets the system identity (locale/time/hostname/users) and ensures your initramfs includes the bits needed to boot from Btrfs.

### 5.1 Pacman tweaks (again, now inside the chroot)
```bash
conf=/etc/pacman.conf
# Same pacman tweaks as on the ISO: raise ParallelDownloads and enable colorized output.
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

### 5.2 Locale, time, and console
```bash
ln -sf /usr/share/zoneinfo/Asia/Bangkok /etc/localtime
hwclock --systohc
# Uncomment desired locales in `/etc/locale.gen` (add or remove lines as appropriate for your setup).
sed -i 's/^#ja_JP.UTF-8 UTF-8/ja_JP.UTF-8 UTF-8/' /etc/locale.gen   # Japanese
sed -i 's/^#th_TH.UTF-8 UTF-8/th_TH.UTF-8 UTF-8/' /etc/locale.gen   # Thai
sed -i 's/^#en_GB.UTF-8 UTF-8/en_GB.UTF-8 UTF-8/' /etc/locale.gen   # British English
sed -i 's/^#en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen   # US English
locale-gen
echo 'LANG=en_US.UTF-8' > /etc/locale.conf
```

### 5.3 Hostname and hosts file
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

### 5.4 Users and sudo
```bash
echo "Setting Root Password"
passwd
```
```bash
user=pop # change username
echo "Create user, and set password"
useradd -m -G wheel,storage,power,audio,video -s /bin/bash $user
passwd $user
```
Enable wheel sudo in `sudoers`
```bash
# uncomment: %wheel ALL=(ALL) ALL
EDITOR=vim visudo   
```

### 5.5 mkinitcpio
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

---

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
    CMDLINE: loglevel=3 root=UUID=${root_uuid} rootflags=subvol=@ rootfstype=btrfs rw resume=UUID=${swap_uuid} zswap.enabled=1 nvidia-drm.modeset=1 nvidia-drm.fbdev=1 ${ucode_img}_iommu=on iommu=pt

/Arch Linux (fallback)
    PROTOCOL: linux
    KERNEL_PATH: boot():/limine/vmlinuz-linux
    MODULE_PATH: boot():/limine/initramfs-linux.img
    CMDLINE: loglevel=3 root=UUID=${root_uuid} rootflags=subvol=@ rootfstype=btrfs rw
EOF

cat /boot/limine/limine.conf
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

---

## Services & QoL
⚙️ Goal: enable the essential background services you want on every boot (networking, printing, bluetooth, etc.). Pick a networking stack and enable only what you actually use to avoid service conflicts.

🧠 Networking note (choose one approach):
- `NetworkManager` (common on desktops/laptops): enable `NetworkManager` (and optionally `iwd` for Wi‑Fi backend).
- `systemd-networkd` + `systemd-resolved` (common on servers/minimal): enable those, skip `NetworkManager` and `dhcpcd`.

### Extra packages (optional)
```bash
pacman -Syu --needed \
  util-linux inetutils usbutils rsync htop bat zip unzip p7zip \   # core system tools & CLI utilities
  iwd \                                                             # wifi
  avahi nss-mdns \                                                  # mDNS
  dhcpcd \                                                          # DHCP, mDNS
  alsa-utils sof-firmware easyeffects \                             # audio tools & firmware
  bluez bluez-utils \                                               # Bluetooth support
  cups \                                                            # printing system
  acpi acpi_call acpid \                                            # power & hardware management
  xdg-user-dirs \                                                   # user directory structure
```

<details>
  <summary>⚙️ Packages being installed (Services &amp; QoL extras)</summary>

  - Basics: `htop`, `inetutils`, `usbutils`, `rsync`, `util-linux`
  - Archives: `zip`, `unzip`, `p7zip`
  - CLI quality-of-life: `bat`
  - Networking (pick what you use): `iwd`, `dhcpcd`, `avahi`, `nss-mdns`
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
# systemctl enable dhcpcd

# Everything else (pick what you need)
systemctl enable bluetooth
systemctl enable cups
systemctl enable avahi-daemon
systemctl enable acpid
systemctl enable reflector.timer
systemctl enable sshd
systemctl enable fstrim.timer
```
---

## Desktop Stack
🖥️ Goal: install KDE Plasma + a display manager (SDDM) and the integration pieces you’ll want on a typical desktop (portals, NetworkManager applet, audio, thumbnails). If you don’t want many of the KDE apps, you can skip the Desktop Apps section.

### KDE Plasma & apps
#### KDE Core
```bash
pacman -S --needed \
  sddm                    # Display/login manager (graphical login screen)
  sddm-kcm                # System Settings module to configure SDDM
  xdg-desktop-portal      # “Portal” framework (file picker, screen share, sandbox app integration)
  xdg-desktop-portal-kde  # KDE backend for portals (needed for Wayland screen share, Flatpak, etc.)
  qt6-wayland             # Qt6 Wayland platform plugin
  xorg-xwayland           # Runs X11 apps under Wayland

systemctl enable sddm
systemctl enable power-profiles-daemon
```

#### KDE Plasma Core
```bash
pacman -S --needed \
  plasma-desktop          # The Plasma desktop shell (panels, launcher, desktop UI)
  plasma-workspace        # Core workspace components (session bits, shell integration, essentials)
  plasma-wayland-session  # Wayland session entry + components for logging into Plasma Wayland
  kwin                    # KDE window manager + compositor (Wayland/X11)
  systemsettings          # KDE System Settings app
  plasma-nm               # NetworkManager integration (network tray, VPN UI)
  plasma-pa               # Audio volume controls for PipeWire/PulseAudio
  kscreen                 # Display configuration + monitor hotplug handling
  kde-gtk-config          # Configure GTK theme/fonts under KDE
  breeze-gtk              # Breeze theme for GTK apps (visual consistency)

```

#### KDE Plasma (Optionals)
You can select what you need.
```bash
pacman -S --needed bluedevil                    # Bluetooth tray + pairing UI
pacman -S --needed power-profiles-daemon        # Laptop power modes (balanced/performance/powersave)
pacman -S --needed kdeplasma-addons             # Extra Plasma widgets/applets (more features, more stuff)
# pacman -S --needed plasma-vault                 # Encrypted “vault” folders integration
pacman -S --needed plasma-systemmonitor         # KDE System Monitor app (optional if you use htop/Mission Center)
pacman -S --needed plasma-browser-integration   # Browser media controls + integration
pacman -S --needed discover                     # KDE software center (can add background notifier)
pacman -S --needed krdp                         # KDE Remote Desktop server/client bits (krdpserver)
pacman -S --needed print-manager                # KDE printer management UI
# pacman -S --needed kdeconnect                   # Phone integration (kdeconnectd service)
```

#### Desktop Apps
````bash
pacman -S --needed dolphin                    # File manager
pacman -S --needed dolphin-plugins            # Git/Share/extra Dolphin integrations
pacman -S --needed konsole                    # Terminal
pacman -S --needed kate                       # GUI text editor
pacman -S --needed okular                     # PDF/EPUB viewer
pacman -S --needed gwenview                   # Image viewer
pacman -S --needed spectacle                  # Screenshot tool
pacman -S --needed ark                        # Archive manager GUI
pacman -S --needed kio-extras                 # SMB/SFTP/etc. support inside Dolphin/KIO
pacman -S --needed ffmpegthumbs               # Video thumbnails in Dolphin
pacman -S --needed kdegraphics-thumbnailers   # Document/image thumbnail plugins
pacman -S --needed filelight                  # Disk usage GUI
pacman -S --needed kcalc                      # Calculator

````

<details>
  <summary>🖥️ Packages being installed (KDE Plasma desktop)</summary>

  - Display manager: `sddm`, `sddm-kcm`
  - Portals/Wayland helpers: `xdg-desktop-portal`, `xdg-desktop-portal-kde`, `qt6-wayland`, `xorg-xwayland`
  - Plasma core: `plasma-desktop`, `plasma-workspace`, `plasma-wayland-session`, `kwin`, `systemsettings`
  - Plasma integration: `plasma-nm`, `plasma-pa`, `kscreen`, `kde-gtk-config`, `breeze-gtk`
  - Optional Plasma extras: `bluedevil`, `power-profiles-daemon`, `kdeplasma-addons`, `plasma-systemmonitor`, `plasma-browser-integration`, `discover`, `krdp`, `print-manager`
  - Desktop apps: `dolphin`, `dolphin-plugins`, `konsole`, `kate`, `okular`, `gwenview`, `spectacle`, `ark`, `filelight`, `kcalc`
  - Thumbnails/integration: `kio-extras`, `ffmpegthumbs`, `kdegraphics-thumbnailers`
</details>

---

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

---

## Login to your new system using your user account

### YAY package manager
```bash
sudo pacman -S --needed git base-devel && git clone https://aur.archlinux.org/yay.git && cd yay && makepkg -si
```

### Extra Packages for Limine
```bash
yay -S --needed \
btrfs-assistant snapper-gui-git snapper-tools
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
# Optional (AUR): keep Limine + initramfs in sync with snapshots/updates
yay -S limine-snapper-sync limine-mkinitcpio-hook
sudo limine-snapper-sync 

sudo pacman -Syu snap-pac
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
sudo snapper -c root create-config /
sudo snapper -c home create-config /home
sudo sed -i 's/^TIMELINE_CREATE="yes"/TIMELINE_CREATE="no"/' /etc/snapper/configs/{root,home}           # disable automatic timeline snapshots
sudo sed -i 's/^NUMBER_LIMIT="50"/NUMBER_LIMIT="5"/' /etc/snapper/configs/{root,home}                  # keep only 5 regular snapshots
sudo sed -i 's/^NUMBER_LIMIT_IMPORTANT="10"/NUMBER_LIMIT_IMPORTANT="5"/' /etc/snapper/configs/{root,home} # keep only 5 important snapshots

cp /etc/limine-snapper-sync.conf /etc/default/limine
```

---

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
sudo pacman -S \
  linux-cachyos-eevdf linux-cachyos-eevdf-headers \   # CachyOS EEVDF kernel + headers
  linux-cachyos linux-cachyos-headers                 # Default CachyOS kernel + headers
```

<details>
  <summary>🐆 Packages being installed (CachyOS kernels)</summary>

  - `linux-cachyos`, `linux-cachyos-headers`: CachyOS kernel + headers
  - `linux-cachyos-eevdf`, `linux-cachyos-eevdf-headers`: alternative CachyOS kernel variant + headers
</details>

CachyOS Packages
```bash
sudo pacman -S \
  cachyos-settings \           # CachyOS defaults/tweaks
  appmenu-gtk-module \         # AppMenu GTK module
  libdbusmenu-glib \           # DBus menu integration
  cachyos-gaming-meta \        # Gaming-oriented meta package
  cachyos-hello                # Welcome/info app
```

<details>
  <summary>🐆 Packages being installed (CachyOS extras)</summary>

  - `cachyos-settings`: CachyOS defaults/tweaks
  - `cachyos-gaming-meta`: gaming-oriented meta package
  - `cachyos-hello`: welcome/info app
  - `appmenu-gtk-module`, `libdbusmenu-glib`: appmenu/DBus menu integration
</details>

Update all packages to CachyOS Optimized
```bash
sudo pacman -Qqn | sudo pacman -S -
```

### Extra Packages and fonts
🧺 Personal “daily driver” package list. Treat it as a pick-list; some items overlap with earlier sections and some are heavy (office suite, IDE, Steam).
```bash
## Core CLI + utilities
yay -S --needed \
  openbsd-netcat     # nc: TCP/UDP swiss-army knife \
  imagemagick        # image convert/resize/identify CLI tools

## Filesystem / network integration
yay -S --needed \
  gvfs               # virtual filesystem layer (trash, mtp, gphoto2, etc.) \
  gvfs-smb           # SMB/CIFS browsing in Dolphin (Windows shares)

## Power / laptop bits
yay -S --needed \
  brightnessctl      # backlight/brightness control (laptops)

## Audio / media / creative
yay -S --needed \
  vlc                # media player (handles basically everything) \
  gimp               # image editor \
  obs-studio         # screen recording + streaming

## Desktop apps
yay -S --needed \
  firefox            # web browser \
  libreoffice-fresh  # office suite (latest branch; big but useful) \
  mailspring         # email client (Electron-based) \
  visual-studio-code-bin # VS Code (official build; AUR)

## Gaming stack
yay -S --needed \
  gamemode              # Feral gamemode (CPU governor/priority tweaks) \
  steam                 # Steam client \
  lutris                # game launcher (Wine/Proton management) \
  mangohud              # Vulkan/OpenGL performance overlay \
  goverlay              # GUI to configure MangoHud \
  proton-ge-custom-bin  # Proton-GE builds (AUR) for better game compatibility

## Flatpak
yay -S --needed \
  flatpak               # Flatpak runtime + app management

## Fonts
yay -S --needed \
  noto-fonts                    # Google Noto base fonts \
  noto-fonts-cjk                # Noto for Chinese/Japanese/Korean \
  noto-fonts-emoji              # Noto Color Emoji \
  noto-fonts-extra              # extra Noto families \
  ttf-dejavu                    # solid fallback font set \
  ttf-liberation                # metric-compatible with Arial/Times/Courier \
  ttf-jetbrains-mono            # dev-friendly monospace \
  ttf-fira-code                 # monospace with ligatures \
  ttf-ubuntu-font-family        # Ubuntu UI font family \
  terminus-font                 # crisp bitmap-ish console font \
  adobe-source-sans-fonts       # Adobe Source Sans \
  adobe-source-serif-fonts      # Adobe Source Serif \
  adobe-source-code-pro-fonts   # Adobe Source Code Pro \
  nerd-fonts                    # icon glyphs patched into many fonts (large install) \
  ttf-ms-fonts                  # Microsoft core fonts (AUR; licensing caveats)

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

### Nvidia Driver
🟩 This is a rough checklist for an NVIDIA DKMS setup. Exact package names and kernel module steps depend on your GPU generation and kernel choice, so verify against the Arch Wiki for your hardware.

```bash
yay -S --needed \
  nvidia-dkms         # NVIDIA DKMS driver (kernel modules) \
  nvidia-utils        # NVIDIA userspace libraries + tools \
  lib32-nvidia-utils  # 32-bit NVIDIA libs (Steam/Proton) \
  nvidia-settings     # NVIDIA X11 settings GUI \
  ocl-icd             # OpenCL ICD loader \
  opencl-nvidia       # NVIDIA OpenCL implementation \
  clinfo              # query OpenCL platforms/devices \
  cuda                # CUDA toolkit/runtime
```

<details>
  <summary>🟩 Packages being installed (NVIDIA DKMS + CUDA/OpenCL)</summary>

  - Driver (DKMS): `nvidia-dkms`, `nvidia-utils`, `lib32-nvidia-utils`, `nvidia-settings`
  - OpenCL: `ocl-icd`, `opencl-nvidia`, `clinfo`
  - CUDA toolkit/runtime: `cuda`
</details>

#### Add DRM kernel module

Use Perl to ensure these kernel parameters are present in Limine `CMDLINE` (idempotent for both `/boot/limine/limine.conf` and `/boot/limine.conf` if it exists):
```bash
for conf in /boot/limine/limine.conf /boot/limine.conf; do
  [ -f "$conf" ] || continue                                   # skip if file does not exist
  sudo perl -pi -e '
    s/^CMDLINE:\s*(.*)$/do {
      my $line  = $1;
      my $extra = " nvidia-drm.modeset=1 nvidia-drm.fbdev=1";  # flags to append
      # Only append if both flags are not already present
      $line .= $extra unless $line =~ /\bnvidia-drm\.modeset=1\b/ && $line =~ /\bnvidia-drm\.fbdev=1\b/;
      "CMDLINE: $line"
    }/e;
  ' "$conf"
done
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
sudo mkdir -p /etc/pacman.d/hooks/                             # ensure hooks directory exists
sudo tee /etc/pacman.d/hooks/nvidia.hook >/dev/null <<'EOF'    # hook: rebuild NVIDIA initramfs on updates
[Trigger]
Operation = Install
Operation = Upgrade
Operation = Remove
Type = Package
Target = nvidia-dkms
Target = linux-cachyos
Target = linux-cachyos-eevdf
Target = linux
# Adjust line(6) above to match your driver, e.g. Target=nvidia-470xx-dkms
# Change line(7) above, if you are not using the regular kernel For example, Target=linux-lts

[Action]
Description = Update Nvidia module in initcpio
Depends = mkinitcpio
When = PostTransaction
NeedsTargets
Exec = /bin/sh -c 'while read -r trg; do case $trg in linux) exit 0; esac; done; /usr/bin/mkinitcpio -P'
EOF
```

Disable annoying kwallet
```bash
mkdir -p ~/.config
cat <<'EOF' >> ~/.config/kwalletrc
[Wallet]
Enabled=false
EOF
```

---

## Fish terminal
🎣 Goal: install the Fish shell, make it your default interactive shell, and drop in a small starter config you can edit later.

### Install Fish and make it your login shell
```bash
# Install Fish from the official repos
sudo pacman -S --needed fish

# Ensure fish is listed as a valid login shell (only appends if missing)
grep -qxF /usr/bin/fish /etc/shells || echo /usr/bin/fish | sudo tee -a /etc/shells

# Make fish the default shell for your current user
chsh -s /usr/bin/fish "$USER"
```

### Minimal Fish config (optional)
```bash
mkdir -p ~/.config/fish
cat <<'EOF' > ~/.config/fish/config.fish
# Example Fish config — adjust to taste.

# Editors/pager
set -gx EDITOR vim
set -gx PAGER less

# Key bindings (comment this out if you prefer default bindings)
fish_vi_key_bindings

# Common aliases
alias ll='ls -lh --color=auto'
alias la='ls -A'
alias grep='grep --color=auto'
alias fgrep='fgrep --color=auto'
alias egrep='egrep --color=auto'

# Fasrfetch, because
fastfetch
EOF
```

After this, open a new terminal or log out/log back in to start using Fish by default.

---

## pyenv
🐍 Goal: install `pyenv` so you can manage multiple Python versions per‑user, and wire it into your shell (Fish in this example).

### Install build dependencies and pyenv
```bash
# Install common build deps for compiling Python versions
sudo pacman -S --needed base-devel openssl zlib xz tk readline sqlite libffi bzip2

# Clone pyenv into your home directory
git clone https://github.com/pyenv/pyenv.git ~/.pyenv
```

### Shell integration (Fish)
```bash
# Add to ~/.config/fish/config.fish (below your other exports/aliases)
set -gx PYENV_ROOT $HOME/.pyenv
fish_add_path $PYENV_ROOT/bin

# Initialize pyenv on every new shell
pyenv init - | source
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

---

## Credits & Thanks
🙏 Huge thanks to the authors/maintainers of these guides and notes that helped shape parts of this memo:
- `fstab` notes: https://gist.github.com/mjkstra/96ce7a5689d753e7a6bdd92cdc169bae#fstab
- Arch install walkthrough/reference: https://github.com/silentz/arch-linux-install-guide
- Chroot exit + reboot checklist: https://gist.github.com/yovko/512326b904d120f3280c163abfbcb787#exit-from-chroot-and-reboot
- NVIDIA driver guide: https://github.com/korvahannu/arch-nvidia-drivers-installation-guide
