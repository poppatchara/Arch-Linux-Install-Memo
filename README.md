# Arch Linux + Btrfs + CachyOS Kernels + KDE Plasma + Limine

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
sudo sed -i \
  -e 's/^\(ParallelDownloads *= *\)5/\115/' \
  -e 's/^#Color/Color/' \
  "$conf"

# Uncomment [multilib] and its Include line (assumed to be near the end of the file).
lines="$(wc -l < "$conf")"
if [ "$lines" -ge 8 ]; then
  sudo sed -i \
    -e "$((lines-8)) s/^#//" \
    -e "$((lines-7)) s/^#//" \
    -e "$((lines-6)) s/^#//" \
    "$conf"
fi
```

### 0.2 Refresh mirror list 🌐
Reorder the existing mirrorlist: move your `## <country>` section to right before `## Worldwide`, with exactly one blank line between them.
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
```
```bash
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
```
```bash
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
  base base-devel linux linux-headers linux-firmware btrfs-progs \
  networkmanager vim nvim git sudo "${ucode_pkg}" man curl \
  efibootmgr inotify-tools pipewire pipewire-alsa pipewire-pulse pipewire-jack \
  wireplumber reflector zsh zsh-completions zsh-autosuggestions \
  bash-completion openssh

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
  - Editors + tools: `vim`, `nvim`, `git`, `sudo`, `man`, `curl`, `inotify-tools`
  - Audio stack (PipeWire): `pipewire`, `pipewire-alsa`, `pipewire-pulse`, `pipewire-jack`, `wireplumber`
  - Mirrors/shell UX: `reflector`, `zsh`, `zsh-completions`, `zsh-autosuggestions`, `bash-completion`
</details>

Wait for the install to finish, then continue inside the chroot.

## Chroot Configuration
🏠 Everything below runs inside `arch-chroot /mnt` unless explicitly stated otherwise. This section sets the system identity (locale/time/hostname/users) and ensures your initramfs includes the bits needed to boot from Btrfs.

### 5.1 Pacman tweaks (again, now inside the chroot)
```bash
conf=/etc/pacman.conf
sed -i \
  -e 's/^\(ParallelDownloads *= *\)5/\115/' \
  -e 's/^#Color/Color/' \
  "$conf"

# Uncomment [multilib] and its Include line (assumed to be near the end of the file).
lines="$(wc -l < "$conf")"
if [ "$lines" -ge 8 ]; then
  sed -i \
    -e "$((lines-8)) s/^#//" \
    -e "$((lines-7)) s/^#//" \
    -e "$((lines-6)) s/^#//" \
    "$conf"
fi
pacman -Syy
```

### 5.2 Locale, time, and console
```bash
ln -sf /usr/share/zoneinfo/Asia/Bangkok /etc/localtime
hwclock --systohc
sed -i 's/^#ja_JP.UTF-8 UTF-8/ja_JP.UTF-8 UTF-8/' /etc/locale.gen
sed -i 's/^#th_TH.UTF-8 UTF-8/th_TH.UTF-8 UTF-8/' /etc/locale.gen
sed -i 's/^#en_GB.UTF-8 UTF-8/en_GB.UTF-8 UTF-8/' /etc/locale.gen
sed -i 's/^#en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen
locale-gen
echo 'LANG=en_US.UTF-8' > /etc/locale.conf
```

### 5.3 Hostname and hosts file
```bash
hname="arch"
echo "${hname}" > /etc/hostname
cat <<EOF > /etc/hosts
127.0.0.1   localhost
::1         localhost
127.0.1.1   ${hname}.localdomain ${hname}
EOF
```

### 5.4 Users and sudo
```bash
passwd
```
```bash
useradd -m -G wheel,storage,power,audio,video -s /bin/bash pop   # change username
passwd pop
```
Enable wheel sudo in `sudoers`
```bash
EDITOR=vim visudo   # uncomment: %wheel ALL=(ALL) ALL
```

### 5.5 mkinitcpio
Look for these lines in `/etc/mkinitcpio.conf` and replace to this.

```ini
MODULES=(btrfs)
...
BINARIES=(/usr/bin/btrfs)
...
HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block encrypt filesystems resume fsck)
...
```

Then rebuild initramfs:
```bash
mkinitcpio -P
```

---

## Limine Bootloader
🚀 Goal: install Limine to the EFI System Partition and generate a `limine.conf` that boots your Arch kernel/initramfs from Btrfs by UUID. The key idea is to keep Limine’s config and the kernel artifacts together under `/boot/limine`.

### 6.1 Install Limine binaries (ESP mounted at `/boot`)
```bash
pacman -S --needed limine
mkdir -p /boot/EFI/limine /boot/limine
cp -v /usr/share/limine/*.EFI /boot/EFI/limine/
```

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
ucode_img="intel-ucode"
if lscpu | grep -qi amd; then
  ucode_img="amd-ucode"
fi

cat <<EOF | tee /boot/limine/limine.conf >/dev/null
TIMEOUT=3
DEFAULT_ENTRY=Arch Linux

/Arch Linux
    PROTOCOL: linux
    KERNEL_PATH: boot():/limine/vmlinuz-linux
    MODULE_PATH: boot():/limine/${ucode_img}.img
    MODULE_PATH: boot():/limine/initramfs-linux.img
    CMDLINE: loglevel=3 root=UUID=${root_uuid} rootflags=subvol=@ rootfstype=btrfs rw resume=UUID=${swap_uuid} zswap.enabled=1 nvidia-drm.modeset=1 nvidia-drm.fbdev=1 amd_iommu=on iommu=pt

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
pacman -Syu wget htop inetutils imagemagick usbutils easyeffects nss-mdns bat zip unzip \
p7zip xdg-user-dirs noto-fonts nerd-fonts ttf-jetbrains-mono libreoffice-fresh \
sof-firmware bluez bluez-utils cups util-linux terminus-font openssh rsync \
dhcpcd avahi acpi acpi_call acpid alsa-utils pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber iwd
```

<details>
  <summary>⚙️ Packages being installed (Services &amp; QoL extras)</summary>

  - Basics: `wget`, `htop`, `inetutils`, `usbutils`, `rsync`
  - Archives: `zip`, `unzip`, `p7zip`
  - CLI quality-of-life: `bat`
  - Images/media: `imagemagick`
  - Audio firmware/tools: `sof-firmware`, `alsa-utils`, `easyeffects`
  - PipeWire stack: `pipewire`, `pipewire-alsa`, `pipewire-pulse`, `pipewire-jack`, `wireplumber`
  - Networking (pick what you use): `iwd`, `dhcpcd`, `avahi`, `nss-mdns`, `openssh`
  - Bluetooth: `bluez`, `bluez-utils`
  - Printing: `cups`
  - Power/ACPI: `acpi`, `acpi_call`, `acpid`
  - Fonts/terminal: `noto-fonts`, `nerd-fonts`, `ttf-jetbrains-mono`, `terminus-font`
  - Desktop/user dirs: `xdg-user-dirs`
  - System utilities: `util-linux`
  - Office: `libreoffice-fresh`
</details>

### Enable services
```bash
# Networking (choose one)
systemctl enable NetworkManager
# systemctl enable systemd-networkd
# systemctl enable systemd-resolved

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
```

### Periodic TRIM
```bash
pacman -S --needed util-linux
systemctl enable fstrim.timer
```

<details>
  <summary>🧹 Packages being installed (Periodic TRIM)</summary>

  - `util-linux`: provides `fstrim` and `fstrim.timer`
</details>

---

## Desktop Stack
🖥️ Goal: install KDE Plasma + a display manager (SDDM) and the integration pieces you’ll want on a typical desktop (portals, NetworkManager applet, audio, thumbnails). If you don’t want the full KDE app suite, replace `kde-applications-meta` with a hand-picked list.

### KDE Plasma & apps
```bash
pacman -S --needed \
  plasma-meta kde-applications-meta \
  sddm sddm-kcm \
  plasma-nm plasma-pa kscreen bluedevil print-manager \
  xdg-desktop-portal xdg-desktop-portal-kde \
  dolphin dolphin-plugins konsole kate \
  okular gwenview spectacle ark power-profiles-daemon \
  kdeconnect kio-extras ffmpegthumbs kdegraphics-thumbnailers \
  filelight kcalc \
  noto-fonts noto-fonts-cjk noto-fonts-emoji ttf-dejavu ttf-liberation \
  ttf-jetbrains-mono ttf-fira-code ttf-ubuntu-font-family \
  adobe-source-sans-fonts adobe-source-serif-fonts adobe-source-code-pro-fonts

systemctl enable sddm
systemctl enable power-profiles-daemon
```

<details>
  <summary>🖥️ Packages being installed (KDE Plasma desktop)</summary>

  - Plasma desktop: `plasma-meta`
  - KDE apps bundle: `kde-applications-meta` (large meta package)
  - Display manager: `sddm`, `sddm-kcm`
  - Desktop integration: `plasma-nm`, `plasma-pa`, `kscreen`, `bluedevil`, `print-manager`
  - Portals: `xdg-desktop-portal`, `xdg-desktop-portal-kde`
  - Common apps: `dolphin`, `dolphin-plugins`, `konsole`, `kate`, `okular`, `gwenview`, `spectacle`, `ark`, `filelight`, `kcalc`
  - Thumbnails/integration: `kdeconnect`, `kio-extras`, `ffmpegthumbs`, `kdegraphics-thumbnailers`
  - Power profiles: `power-profiles-daemon`
  - Fonts: `noto-fonts`, `noto-fonts-cjk`, `noto-fonts-emoji`, `ttf-dejavu`, `ttf-liberation`, `ttf-jetbrains-mono`, `ttf-fira-code`, `ttf-ubuntu-font-family`, `adobe-source-sans-fonts`, `adobe-source-serif-fonts`, `adobe-source-code-pro-fonts`
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

### Snapper
📸 Goal: set up snapshot management for Btrfs so you can roll back system changes. `snapper` itself is in the official repos; the Limine integration shown below uses AUR packages (requires an AUR helper such as `yay`, see Post-Install Ideas).

yay
```bash
sudo pacman -S --needed git base-devel && git clone https://aur.archlinux.org/yay.git && cd yay && makepkg -si
```

<details>
  <summary>🧰 Packages being installed (bootstrap AUR helper)</summary>

  - `git`: fetch the `yay` PKGBUILD repo
  - `base-devel`: required toolchain for building AUR packages via `makepkg`
</details>

snapper
```bash
pacman -Syu snapper
# Optional (AUR): keep Limine + initramfs in sync with snapshots/updates
yay -S limine-snapper-sync limine-mkinitcpio-hook
```

<details>
  <summary>📸 Packages being installed (Snapper + optional Limine integration)</summary>

  - `snapper`: manage Btrfs snapshots (create/list/rollback policies)
  - (AUR) `limine-snapper-sync`: keep Limine config/entries aligned with snapshots
  - (AUR) `limine-mkinitcpio-hook`: mkinitcpio hook for Limine workflows
</details>

Create configs
```bash
snapper -c root create-config /
snapper -c home create-config /home
sudo sed -i 's/^TIMELINE_CREATE="yes"/TIMELINE_CREATE="no"/' /etc/snapper/configs/{root,home}
sudo sed -i 's/^NUMBER_LIMIT="50"/NUMBER_LIMIT="5"/' /etc/snapper/configs/{root,home}
sudo sed -i 's/^NUMBER_LIMIT_IMPORTANT="10"/NUMBER_LIMIT_IMPORTANT="5"/' /etc/snapper/configs/{root,home}

cp /etc/limine-snapper-sync.conf /etc/default/limine
pacman -Syu snap-pac
```

<details>
  <summary>📸 Packages being installed (Snapper add-on)</summary>

  - `snap-pac`: pacman hooks to auto-create snapshots around package transactions
</details>

---

### CachyOS Kernels
🐆 If you want CachyOS’ tuned kernels/userspace, add their repo and install the kernel packages. This is optional and changes your system away from “pure Arch”, so it’s a good idea to keep a package list backup first.

Backup pacman config
```bash
sudo cp -a /etc/pacman.conf /etc/pacman.conf.pre-cachy
pacman -Qqe > ~/pkglist.pre-cachy.txt
```

Add CachyOS repos (fast/automated way)

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
  linux-cachyos-eevdf linux-cachyos-eevdf-headers \
  linux-cachyos linux-cachyos-headers
```

<details>
  <summary>🐆 Packages being installed (CachyOS kernels)</summary>

  - `linux-cachyos`, `linux-cachyos-headers`: CachyOS kernel + headers
  - `linux-cachyos-eevdf`, `linux-cachyos-eevdf-headers`: alternative CachyOS kernel variant + headers
</details>

CachyOS Packages
```bash
sudo pacman -S cachyos-settings appmenu-gtk-module libdbusmenu-glib cachyos-gaming-meta cachyos-hello
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
yay -Syu wget htop gvfs gvfs-smb inetutils imagemagick usbutils easyeffects openbsd-netcat nss-mdns bat zip unzip \
p7zip brightnessctl xdg-user-dirs noto-fonts nerd-fonts ttf-jetbrains-mono libreoffice-fresh firefox mailspring \
vlc gimp obs-studio btrfs-progs acpi acpi_call tlp tlp-rdw flatpak gamemode steam lutris mangohud \
visual-studio-code-bin proton-ge-custom-bin goverlay noto-fonts noto-fonts-emoji noto-fonts-extra \
ttf-ms-fonts
```

<details>
  <summary>🧺 Packages being installed (daily-driver pick list)</summary>

  - Desktop integration: `gvfs`, `gvfs-smb`
  - Browsers/mail: `firefox`, `mailspring`
  - Media/creative: `vlc`, `gimp`, `obs-studio`, `imagemagick`
  - Gaming: `steam`, `lutris`, `gamemode`, `mangohud`, `proton-ge-custom-bin`, `goverlay`
  - System/power: `tlp`, `tlp-rdw`, `brightnessctl`, `acpi`, `acpi_call`
  - Filesystem: `btrfs-progs`
  - Networking/tools: `openbsd-netcat`, `inetutils`, `usbutils`, `nss-mdns`
  - Audio: `easyeffects`
  - Packaging/platforms: `flatpak`
  - Editors/IDE: `visual-studio-code-bin`
  - Utilities: `wget`, `htop`, `bat`, `zip`, `unzip`, `p7zip`, `xdg-user-dirs`
  - Fonts: `noto-fonts`, `noto-fonts-emoji`, `noto-fonts-extra`, `nerd-fonts`, `ttf-jetbrains-mono`, `ttf-ms-fonts`
  - Office: `libreoffice-fresh`
</details>

### Nvidia Driver
🟩 This is a rough checklist for an NVIDIA DKMS setup. Exact package names and kernel module steps depend on your GPU generation and kernel choice, so verify against the Arch Wiki for your hardware.

```bash
yay -S nvidia-dkms nvidia-utils lib32-nvidia-utils nvidia-settings \
ocl-icd opencl-nvidia clinfo cuda 
```

<details>
  <summary>🟩 Packages being installed (NVIDIA DKMS + CUDA/OpenCL)</summary>

  - Driver (DKMS): `nvidia-dkms`, `nvidia-utils`, `lib32-nvidia-utils`, `nvidia-settings`
  - OpenCL: `ocl-icd`, `opencl-nvidia`, `clinfo`
  - CUDA toolkit/runtime: `cuda`
</details>

Add DRM Kernel Module

add to `cmdline` in limine
```bash
nvidia-drm.modeset=1 nvidia-drm.fbdev=1
```
in `/etc/mkinitcpio.conf`
```bash
MODULES=(btrfs nvidia nvidia_modeset nvidia_uvm nvidia_drm)
...
remove `kms` from HOOKS
```

```bash
sudo mkdir -p /etc/pacman.d/hooks/
sudo tee /etc/pacman.d/hooks/nvidia.hook >/dev/null <<'EOF'
[Trigger]
Operation=Install
Operation=Upgrade
Operation=Remove
Type=Package
Target=nvidia-dkms
Target=linux-cachyos
Target=linux-cachyos-eevdf
Target=linux
# Adjust line(6) above to match your driver, e.g. Target=nvidia-470xx-dkms
# Change line(7) above, if you are not using the regular kernel For example, Target=linux-lts

[Action]
Description=Update Nvidia module in initcpio
Depends=mkinitcpio
When=PostTransaction
NeedsTargets
Exec=/bin/sh -c 'while read -r trg; do case $trg in linux) exit 0; esac; done; /usr/bin/mkinitcpio -P'
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

## Credits & Thanks
🙏 Huge thanks to the authors/maintainers of these guides and notes that helped shape parts of this memo:
- `fstab` notes: https://gist.github.com/mjkstra/96ce7a5689d753e7a6bdd92cdc169bae#fstab
- Arch install walkthrough/reference: https://github.com/silentz/arch-linux-install-guide
- Chroot exit + reboot checklist: https://gist.github.com/yovko/512326b904d120f3280c163abfbcb787#exit-from-chroot-and-reboot
- NVIDIA driver guide: https://github.com/korvahannu/arch-nvidia-drivers-installation-guide
