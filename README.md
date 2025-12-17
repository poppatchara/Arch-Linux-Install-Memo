# Arch Linux + Btrfs + KDE Plasma + Limine

Personal notes for reproducing my preferred Arch Linux installation. The flow starts from a fresh Arch ISO, targets a modern UEFI system, uses a Btrfs root on NVMe, installs KDE Plasma as the desktop, and boots with Limine.

## Goals & Assumptions
- UEFI firmware with Secure Boot disabled.
- Single NVMe drive at `/dev/nvme0n1`.
- Wired network or reliable Wi-Fi with `iwctl`.
- Timezone `America/New_York` (adjust when running commands).
- Hostname `aurora` (change freely).

## 1. Prep Work (on another machine)
1. Download latest Arch ISO and verify checksum:
   ```bash
   curl -O https://mirror.rackspace.com/archlinux/iso/latest/archlinux-x86_64.iso
   curl -O https://mirror.rackspace.com/archlinux/iso/latest/archlinux-x86_64.iso.sig
   gpg --keyserver hkps://keys.openpgp.org --recv-keys 0x9741E8AC
   gpg --verify archlinux-x86_64.iso.sig
   ```
2. Write ISO to USB:
   ```bash
   sudo dd if=archlinux-x86_64.iso of=/dev/sdX bs=4M status=progress oflag=sync
   ```

## 2. Boot ISO & Set Up Live Environment
- `loadkeys us` (or `setxkbmap` for console layout).
- Connect to network, e.g. `iwctl station wlan0 connect <ssid>`.
- Sync time: `timedatectl set-ntp true`.
- Confirm disk: `lsblk`.

## 3. Partitioning (GPT)
Target layout:
| Partition | Size | Type | Purpose |
|-----------|------|------|---------|
| `/dev/nvme0n1p1` | 1G | EFI System (type 1) | `/boot` |
| `/dev/nvme0n1p2` | 1G | Linux filesystem | `/boot/limine` |
| `/dev/nvme0n1p3` | Rest | Linux filesystem | Btrfs root |

Commands:
```bash
sgdisk --zap-all /dev/nvme0n1
sgdisk -n1:0:+1G -t1:ef00 -c1:"EFI"
sgdisk -n2:0:+1G -t2:8300 -c2:"Limine"
sgdisk -n3:0:0   -t3:8300 -c3:"Btrfs"
```

## 4. Format & Subvolumes
```bash
mkfs.vfat -n EFI /dev/nvme0n1p1
mkfs.ext4 -L LIMINE /dev/nvme0n1p2
mkfs.btrfs -L archroot /dev/nvme0n1p3

mount /dev/nvme0n1p3 /mnt
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@log
btrfs subvolume create /mnt/@pkg
umount /mnt

mount -o subvol=@,compress=zstd,noatime /dev/nvme0n1p3 /mnt
mkdir -p /mnt/{boot,home,var/log,var/cache/pacman/pkg}
mount -o subvol=@home,compress=zstd,noatime /dev/nvme0n1p3 /mnt/home
mount -o subvol=@log,compress=zstd,noatime /dev/nvme0n1p3 /mnt/var/log
mount -o subvol=@pkg,compress=zstd,noatime /dev/nvme0n1p3 /mnt/var/cache/pacman/pkg
mount /dev/nvme0n1p1 /mnt/boot
mount /dev/nvme0n1p2 /mnt/boot/limine
```

## 5. Base Install
```bash
pacstrap -K /mnt base base-devel linux linux-firmware btrfs-progs \
        networkmanager vim zsh git sudo
genfstab -U /mnt >> /mnt/etc/fstab
arch-chroot /mnt
```

Inside the chroot:
```bash
ln -sf /usr/share/zoneinfo/America/New_York /etc/localtime
hwclock --systohc
sed -i 's/^#en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen
locale-gen
echo 'LANG=en_US.UTF-8' > /etc/locale.conf
echo aurora > /etc/hostname
cat <<EOF > /etc/hosts
127.0.0.1   localhost
::1         localhost
127.0.1.1   aurora.localdomain aurora
EOF
```

Adjust `mkinitcpio.conf`:
- Set `MODULES=(btrfs)`
- Hooks order: `HOOKS=(base systemd autodetect keyboard sd-vconsole modconf block filesystems fsck)`
- Rebuild: `mkinitcpio -P`

Enable networking and sudo:
```bash
systemctl enable NetworkManager
useradd -m -G wheel -s /usr/bin/zsh pop
passwd pop
EDITOR=vim visudo  # uncomment %wheel ALL=(ALL:ALL) ALL
```

## 6. Limine Bootloader
Limine lives on the small ext4 partition.
```bash
pacman -S --needed limine
mkdir -p /boot/limine
limine-deploy /dev/nvme0n1
```

Sample `/boot/limine/limine.conf`:
```
TIMEOUT=5
DEFAULT_ENTRY=Arch Linux

:Arch Linux
    PROTOCOL=linux
    KERNEL_PATH=boot:///vmlinuz-linux
    MODULE_PATH=boot:///initramfs-linux.img
    CMDLINE=root=UUID=<UUID_OF_BTRFS> rootflags=subvol=@ rw loglevel=3 quiet
```

Run `limine-install /boot/limine` after editing the config.

## 7. KDE Plasma & Desktop Stack
```bash
pacman -S --needed plasma-meta plasma-wayland-session sddm \
    pipewire pipewire-pulse wireplumber power-profiles-daemon \
    konsole dolphin firefox
systemctl enable sddm
systemctl enable power-profiles-daemon
```

Optional extras:
- `flatpak` + Flathub remote.
- `steam`, `gamescope`, `mangohud` for gaming.
- `packages: bluez bluez-utils` for Bluetooth; enable `bluetooth.service`.

## 8. Post-Install Polishing
- `systemctl enable fstrim.timer`
- Configure snapshots (e.g., `snapper` or `btrbk`) for Btrfs.
- Set up `reflector` timer for mirrors.
- Install microcode (`intel-ucode` or `amd-ucode`) and add module path in Limine entry.
- `grc` / `bat` / `zoxide` quality-of-life tools.

## 9. Exit, Reboot
```bash
exit
umount -R /mnt
shutdown -r now
```

Future ideas: automate with `archinstall` profile, add scripts for snapshotting, and document disaster recovery using Btrfs send/receive.
