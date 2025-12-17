# Arch Linux + Btrfs + CachyOS Kernels + KDE Plasma + Limine + Dualboot + Nvidia DKMS + Snapper 

Personal notes for reproducing my preferred Arch Linux installation. The flow starts from a fresh Arch ISO, targets a modern UEFI system, uses a Btrfs root on NVMe, installs KDE Plasma as the desktop, and boots with Limine.

## Assumptions
- Assumed you already made a USB media
- I use another machine to ssh in to the live install. It made copy and paste command much easier.
- Just set the root password using `passwd` and check the ip with `ip a`.
- UEFI firmware with Secure Boot disabled.
- Boot NVMe drive at `/dev/nvme0n1`.
- Wired network or reliable Wi-Fi with `iwctl`.
- Timezone `Asia/Bangkok` (adjust when running commands).
- Hostname `arch` (change freely).
- ESP partition already exists, first partition.
- Disk Swap is lask partition. It is easier if you want to resize it later.



### 0. Preconfig the install medis

Set Parallel and Color
```bash
sudo sed -i \
  -e 's/^\(ParallelDownloads *= *\)5/\115/' \
  -e 's/^#Color/Color/' \
  -e 's/^#\[multilib\]/[multilib]/' \
  -e 's|^#Include = /etc/pacman.d/mirrorlist|Include = /etc/pacman.d/mirrorlist|' \
  /etc/pacman.conf
```

Set pacmaan parallel and mirror list. Change the country to your location.
```bash
country=Thailand
mirror_url="https://archlinux.org/mirrorlist/?country=all&protocol=http&protocol=https&ip_version=4"
tmpfile="$(mktemp)"

curl -s "$mirror_url" > "$tmpfile"
awk -v country="$country" '
  NR <= 3 { header = header $0 ORS; next }
  /^## / { current = $0; next }
  current == ("## " country)   { country_block = country_block $0 ORS; next }
  current == "## Worldwide"    { world_block   = world_block   $0 ORS; next }
  { next }
  END {
    printf "%s", header
    printf "## %s\n", country
    printf "%s", country_block
    printf "\n## Worldwide\n"
    printf "%s", world_block
  }
' "$tmpfile" | sed 's/^#Server/Server/' | tee /etc/pacman.d/mirrorlist >/dev/null
rm -f "$tmpfile"
```

## 1. Partitioning (GPT)

Target layout:
| Partition | Size | Type | Purpose |
|-----------|------|------|---------|
| `/dev/nvme0n1p1` | 4G | EFI System (type 1) | `/boot` |
| `/dev/nvme0n1p2` | Rest | Linux filesystem | Btrfs root|
| `/dev/nvme0n1p3` | About Ram size | Linux Swap | swap |

Commands:
```bash
cfdisk
```

## 2. Format

```bash
mkfs.btrfs -f /dev/nvme0n1p2 -L "ArchLinuxFS"
mkswap /dev/nvme0n1p3
```

Check UUIDs of the partitions and save it in vars:
```bash
esp_uuid="$(blkid -s UUID -o value /dev/nvme0n1p1)"
root_uuid="$(blkid -s UUID -o value /dev/nvme0n1p2)"
swap_uuid="$(blkid -s UUID -o value /dev/nvme0n1p3)"
```

## 3. Btrfs Subvolumes

```bash
echo "Mount blank btrfs vol and create subvols"
mount UUID=${root_uuid} /mnt
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@var
btrfs subvolume create /mnt/@var
btrfs subvolume create /mnt/@root
btrfs subvolume create /mnt/@srv
umount /mnt

echo "Mount btrfs and mount subvols to mount points"
mount -o compress=zstd:1,noatime,subvol=@ UUID=${root_uuid}  /mnt

mount --mkdir -o compress=zstd:1,noatime,subvol=@home UUID=${root_uuid}  /mnt/home
mount --mkdir -o compress=zstd:1,noatime,subvol=@log UUID=${root_uuid}  /mnt/var/log
mount --mkdir -o compress=zstd:1,noatime,subvol=@cache UUID=${root_uuid}  /mnt/var/cache
mount --mkdir -o compress=zstd:1,noatime,subvol=@root UUID=${root_uuid}  /mnt/var/root
mount --mkdir -o compress=zstd:1,noatime,subvol=@srv UUID=${root_uuid}  /mnt/var/srv

echo "Mount /boot"
mount --mkdir UUID=${esp_uuid}  /mnt/boot

echo "Swapon"
swaopn UUID=${swap_uuid}

echo "genfstab"
genfstab -U /mnt >> /mnt/etc/fstab
```

## 4. install packages and chroot into /mnt

```bash

ucode_pkg=intel-ucode
if lscpu | grep -qi amd; then
  ucode_pkg=amd-ucode
fi

pacstrap -K /mnt base base-devel linux linux-firmware btrfs-progs \
        networkmanager vim git sudo "${ucode_pkg}" man curl
arch-chroot /mnt
```

## 5. Continue inside chroot

Config pacman inside chroot again
```bash
sudo sed -i \
  -e 's/^\(ParallelDownloads *= *\)5/\115/' \
  -e 's/^#Color/Color/' \
  -e 's/^#\[multilib\]/[multilib]/' \
  -e 's|^#Include = /etc/pacman.d/mirrorlist|Include = /etc/pacman.d/mirrorlist|' \
  /etc/pacman.conf
```

Set Locale
Remove any locale you dont want. Most people only need en_US
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

Hosts and Hostnames. Change hostname to whatever you want.
```bash
hname="arch"
echo $hname~ > /etc/hostname

cat <<EOF > /etc/hosts
127.0.0.1   localhost
::1         localhost
127.0.1.1   ${hname}.localdomain ${hname}
EOF
```

Setup password
```bash
passwd
```

Add user and set passwd
```bash
useradd -m -G wheel,storage,power,audio,video -s /bin/bash <yourusername>
passwd <yourusername>
```

Add wheel group to sudoers file to allow users to run sudo:
```bash
EDITOR=vim visudo
```
[uncomment following line in file]
`%wheel ALL=(ALL) ALL`

Create viconsole
```bash
touch /etc/vconsole.conf
echo KEYMAP=us>>/etc/vconsole.conf
echo FONT=>>/etc/vconsole.conf
```

Adjust `mkinitcpio.conf`:
```bash
sudo sed -i \
  -e 's/^MODULES=.*/MODULES=(btrfs)/' \
  -e 's/^HOOKS=.*/HOOKS=(base udev systemd autodetect microcode modconf keyboard keymap sd-vconsole block filesystems resume fsck)/' \
  /etc/mkinitcpio.conf
mkinitcpio -P
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
