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
8. [Desktop Stack](#desktop-stack)
9. [Services & QoL](#services--qol)
10. [Trim & Reboot](#trim--reboot)
11. [Post-Install Ideas](#post-install-ideas)

---

## Assumptions
- Arch ISO already written to USB (any recent release).
- I SSH into the live ISO from another machine (root password set with `passwd`, IP from `ip a`), purely for copy/paste convenience.
- Secure Boot disabled, UEFI mode enabled.
- Target disk: `/dev/nvme0n1` (adjust commands if yours differs).
- Wired Ethernet or reliable Wi-Fi (`iwctl`) available.
- Timezone `Asia/Bangkok`; hostname defaults to `arch`.
- EFI System Partition (ESP) already exists as the first partition; swap is the last partition to simplify potential resizing.

---

## Live ISO Prep

### 0.1 Configure pacman appearance & parallel downloads
```bash
conf=/etc/pacman.conf
sudo sed -i \
  -e 's/^\(ParallelDownloads *= *\)5/\115/' \
  -e 's/^#Color/Color/' \
  "$conf"

# Uncomment [multilib] and its Include line (assumed to be near the end of the file).
lines="$(wc -l < "$conf")"
sudo sed -i \
  -e "$((lines-6)) s/^#//" \
  -e "$((lines-7)) s/^#//" \
  "$conf"
```

### 0.2 Refresh mirror list
Reorder the existing mirrorlist: move your `## <country>` section to right before `## Worldwide`, with exactly one blank line between them.
```bash
country=Thailand
mirrorfile="/etc/pacman.d/mirrorlist"
sudo cp "$mirrorfile" "${mirrorfile}.bak"

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
' "$mirrorfile" > "$tmpfile" || { echo "mirrorlist reorder failed; restoring backup." >&2; sudo cp "${mirrorfile}.bak" "$mirrorfile"; rm -f "$tmpfile"; exit 1; }

sudo cp "$tmpfile" "$mirrorfile"
rm -f "$tmpfile"

sudo pacman -Syy
```

---

## Partition & Format

### 1. Partition layout
| Partition | Size | Type | Purpose |
|-----------|------|------|---------|
| `/dev/nvme0n1p1` | 4 GiB | EFI System (type `ef00`) | Mounted at `/boot` |
| `/dev/nvme0n1p2` | Remainder | Linux filesystem (`8300`) | Btrfs root |
| `/dev/nvme0n1p3` | ≈ RAM size | Linux swap | Swap |

Use `cfdisk /dev/nvme0n1` (GPT) to build or adjust the layout.

### 2. Format and capture UUIDs
```bash
mkfs.btrfs -f -L ArchLinuxFS /dev/nvme0n1p2
mkswap /dev/nvme0n1p3

esp_uuid="$(blkid -s UUID -o value /dev/nvme0n1p1)"
root_uuid="$(blkid -s UUID -o value /dev/nvme0n1p2)"
swap_uuid="$(blkid -s UUID -o value /dev/nvme0n1p3)"
```

---

## Btrfs Subvolumes & Mounts

```bash
echo "Create subvolumes"
mount UUID="${root_uuid}" /mnt
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@var
btrfs subvolume create /mnt/@log
btrfs subvolume create /mnt/@cache
btrfs subvolume create /mnt/@root
btrfs subvolume create /mnt/@srv
umount /mnt

echo "Mount subvolumes"
mount -o compress=zstd:1,noatime,subvol=@ UUID="${root_uuid}" /mnt
mount --mkdir -o compress=zstd:1,noatime,subvol=@home UUID="${root_uuid}" /mnt/home
mount --mkdir -o compress=zstd:1,noatime,subvol=@var  UUID="${root_uuid}" /mnt/var
mount --mkdir -o compress=zstd:1,noatime,subvol=@log  UUID="${root_uuid}" /mnt/var/log
mount --mkdir -o compress=zstd:1,noatime,subvol=@cache UUID="${root_uuid}" /mnt/var/cache
mount --mkdir -o compress=zstd:1,noatime,subvol=@root UUID="${root_uuid}" /mnt/var/root
mount --mkdir -o compress=zstd:1,noatime,subvol=@srv  UUID="${root_uuid}" /mnt/var/srv

echo "Mount ESP at /boot"
mount --mkdir UUID="${esp_uuid}" /mnt/boot

echo "Enable swap"
swapon UUID="${swap_uuid}"

echo "Generate fstab"
genfstab -U /mnt | tee -a /mnt/etc/fstab
```

---

## Base Install

```bash
ucode_pkg=intel-ucode
if lscpu | grep -qi amd; then
  ucode_pkg=amd-ucode
fi

pacstrap -K /mnt \
  base base-devel linux linux-firmware btrfs-progs \
  networkmanager vim nvim git sudo "${ucode_pkg}" man curl \
  efibootmgr inotify-tools pipewire pipewire-alsa pipewire-pulse pipewire-jack \
  wireplumber reflector zsh zsh-completions zsh-autosuggestions \
  bash-completion openssh

arch-chroot /mnt
```

---

## Chroot Configuration

### 5.1 Pacman tweaks (again, now inside the chroot)
```bash
sed -i \
  -e 's/^\(ParallelDownloads *= *\)5/\115/' \
  -e 's/^#Color/Color/' \
  -e 's/^#\[multilib\]/[multilib]/' \
  -e 's|^#Include = /etc/pacman.d/mirrorlist|Include = /etc/pacman.d/mirrorlist|' \
  /etc/pacman.conf
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

cat <<'EOF' > /etc/vconsole.conf
KEYMAP=us
FONT=
EOF
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
useradd -m -G wheel,storage,power,audio,video -s /bin/bash pop   # change username
passwd pop
EDITOR=vim visudo   # uncomment: %wheel ALL=(ALL) ALL
```

### 5.5 mkinitcpio
```bash
sed -i \
  -e 's/^MODULES=.*/MODULES=(btrfs)/' \
  -e 's/^HOOKS=.*/HOOKS=(base udev systemd autodetect microcode modconf keyboard keymap sd-vconsole block filesystems resume fsck)/' \
  /etc/mkinitcpio.conf
mkinitcpio -P
```

---

## Limine Bootloader

### 6.1 Install Limine binaries (ESP mounted at `/boot`)
```bash
pacman -S --needed limine
mkdir -p /boot/EFI/limine /boot/limine
cp -v /usr/share/limine/*.EFI /boot/EFI/limine/
```

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
ucode_module="boot():/limine/intel-ucode.img"
if lscpu | grep -qi amd; then
  ucode_module="boot():/limine/amd-ucode.img"
fi

cat <<EOF | tee /boot/limine/limine.conf >/dev/null
timeout: 3

:Arch Linux
    PROTOCOL: linux
    KERNEL_PATH: boot():/limine/vmlinuz-linux
    CMDLINE: loglevel=3 root=UUID=${root_uuid} rw rootflags=subvol=@ rootfstype=btrfs
    MODULE_PATH: ${ucode_module}
    MODULE_PATH: boot():/limine/initramfs-linux.img

:Arch Linux (fallback)
    PROTOCOL: linux
    KERNEL_PATH: boot():/limine/vmlinuz-linux
    CMDLINE: loglevel=3 root=UUID=${root_uuid} rw rootflags=subvol=@ rootfstype=btrfs
    MODULE_PATH: ${ucode_module}
    MODULE_PATH: boot():/limine/initramfs-linux-fallback.img
EOF

limine-install /boot/limine
```

### 6.5 Pacman hook to redeploy Limine EFI files
`/etc/pacman.d/hooks/99-limine.hook`
```ini
[Trigger]
Operation = Install
Operation = Upgrade
Type      = Package
Target    = limine

[Action]
Description = Copy Limine EFI files to the ESP
When = PostTransaction
Exec = /usr/bin/cp /usr/share/limine/BOOTX64.EFI /boot/EFI/limine/
```

---

## Desktop Stack

### 7. KDE Plasma (Wayland-first) & friends
```bash
pacman -S --needed \
  plasma-desktop plasma-workspace plasma-wayland-session \
  systemsettings plasma-nm plasma-pa kscreen powerdevil kactivitymanagerd \
  konsole dolphin ark kate kcalc okular spectacle gwenview krdp kdeconnect kio-extras \
  elisa haruna \
  sddm sddm-kcm \
  noto-fonts noto-fonts-cjk noto-fonts-emoji ttf-dejavu ttf-liberation \
  ttf-jetbrains-mono ttf-fira-code ttf-ubuntu-font-family \
  adobe-source-sans-fonts adobe-source-serif-fonts adobe-source-code-pro-fonts

systemctl enable sddm
systemctl enable power-profiles-daemon
```

---

## Services & QoL

```bash
systemctl enable NetworkManager
systemctl enable dhcpcd
systemctl enable iwd
systemctl enable systemd-networkd
systemctl enable systemd-resolved
systemctl enable bluetooth
systemctl enable cups
systemctl enable avahi-daemon
systemctl enable firewalld
systemctl enable acpid
systemctl enable reflector.timer
```

Optional extras:
- `flatpak` + Flathub.
- `steam`, `gamescope`, `mangohud` for gaming.
- `bluez bluez-utils` (already implied by Bluetooth service), `pipewire` stack already installed.
- `grc`, `bat`, `zoxide`, and other quality-of-life CLI tools.
- Snapshot tooling (`snapper`, `btrbk`, etc.) once the system is up.

---

## Trim & Reboot

### 9. Periodic TRIM
```bash
pacman -S --needed util-linux
systemctl enable --now fstrim.timer
```

### 10. Exit chroot and reboot
```bash
exit
umount -R /mnt
swapoff -a
reboot
```

---

## Post-Install Ideas
- Create pacman hooks to sync `/boot/limine` automatically whenever kernels/microcode update.
- Layer `snapper` or `btrbk` for automated Btrfs snapshots, especially before pacman upgrades.
- Add `reflector` configuration (`/etc/xdg/reflector/reflector.conf`) tailored to your region.
- Explore `cachyos` kernels or additional Limine entries for other OSes (dual-boot).
- Document recovery steps (Btrfs send/receive, snapshot rollback) in this repo for quick reference.
