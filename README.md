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
btrfs subvolume create /mnt/@log
btrfs subvolume create /mnt/@cache
btrfs subvolume create /mnt/@root
btrfs subvolume create /mnt/@srv
umount /mnt

echo "Mount btrfs subvolumes"
mount -o compress=zstd:1,noatime,subvol=@ UUID=${root_uuid}  /mnt
mount --mkdir -o compress=zstd:1,noatime,subvol=@home UUID=${root_uuid}  /mnt/home
mount --mkdir -o compress=zstd:1,noatime,subvol=@var UUID=${root_uuid}  /mnt/var
mount --mkdir -o compress=zstd:1,noatime,subvol=@log UUID=${root_uuid}  /mnt/var/log
mount --mkdir -o compress=zstd:1,noatime,subvol=@cache UUID=${root_uuid}  /mnt/var/cache
mount --mkdir -o compress=zstd:1,noatime,subvol=@root UUID=${root_uuid}  /mnt/var/root
mount --mkdir -o compress=zstd:1,noatime,subvol=@srv UUID=${root_uuid}  /mnt/var/srv

echo "Mount ESP at /boot"
mount --mkdir UUID=${esp_uuid}  /mnt/boot

echo "Swapon"
swapon UUID=${swap_uuid}

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
        networkmanager vim nvim git sudo "${ucode_pkg}" man curl \
        efibootmgr inotify-tools pipewire pipewire-alsa pipewire-pulse pipewire-jack \
        wireplumber reflector zsh zsh-completions zsh-autosuggestions \
        bash-completion openssh
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
Limine lives on the FAT32 ESP mounted at `/boot`.
```bash
pacman -S --needed limine

mkdir -p /boot/EFI/limine /boot/limine

cp -v /usr/share/limine/*.EFI /boot/EFI/limine/
```

Now we need to create an entry for Limine in the NVRAM:
```bash
efibootmgr --create --disk /dev/nvme0n1 --part 1 \
      --label "Limine Bootloader" \
      --loader '\EFI\limine\BOOTX64.EFI' \
      --unicode
```
N.B. --disk /dev/nvme0n1 --part 1 means /dev/nvme0n1p1

N.B. Do NOT include boot when pointing to Limine Bootloader file.

N.B. In a Limine config, boot():/ represents the partition on which limine.conf is located.

Copy kernels, initramfs, and microcode into `/boot/limine` so Limine can read them:
```bash
ucode_img="intel-ucode"
if lscpu | grep -qi amd; then
  ucode_img="amd-ucode"
fi

cp -v /boot/vmlinuz-linux /boot/limine/
cp -v /boot/initramfs-linux*.img /boot/limine/
cp -v "/boot/${ucode_img}.img" /boot/limine/
```

Repeat the copy step whenever the kernel or microcode packages update.

Finally let's make a basic configuration for Limine. The following command writes the config directly to `/boot/limine/limine.conf`:

```bash
root_uuid="$(blkid -s UUID -o value /dev/nvme0n1p2)"
ucode_module="boot():/limine/intel-ucode.img"
if lscpu | grep -qi amd; then
  ucode_module="boot():/limine/amd-ucode.img"
fi

cat <<EOF | sudo tee /boot/limine/limine.conf >/dev/null
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

### Pacman hook for Limine
Add `/etc/pacman.d/hooks/99-limine.hook` file with the following content:

```bash
[Trigger]
Operation = Install
Operation = Upgrade
Type = Package
Target = limine

[Action]
Description = Deploying Limine after upgrade...
When = PostTransaction
Exec = /usr/bin/cp /usr/share/limine/BOOTX64.EFI /boot/EFI/limine/
```

### Networking
Enable NetworkManager and systemd-networkd services before rebooting. Otherwise, you won't be able to connect. The systemd-resolved service is a kind of optional, but most probably it is better to enable it. Also you may need dhcpcd.service and (if you need WiFi) iwd.service.

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

## 7. KDE Plasma & Desktop Stack
```bash
sudo pacman -S --needed \
plasma-desktop plasma-workspace plasma-wayland-session \
systemsettings plasma-nm plasma-pa kscreen powerdevil kactivitymanagerd \
konsole dolphin ark kate kcalc okular spectacle gwenview \
krdp kdeconnect kio-extras \
elisa haruna \
sddm sddm-kcm \
noto-fonts noto-fonts-cjk noto-fonts-emoji \
ttf-dejavu ttf-liberation \
ttf-jetbrains-mono ttf-fira-code ttf-ubuntu-font-family \
adobe-source-sans-fonts adobe-source-serif-fonts adobe-source-code-pro-fonts

systemctl enable sddm
systemctl enable power-profiles-daemon
```

### TRIM
```bash
pacman -S --needed util-linux
systemctl enable --now fstrim.timer
```

## 8. Exit from chroot and reboot
It's time to exit the chroot, unmount the /mnt, close the crypted container and reboot to the newly installed Arch Linux.

```bash
exit
umount -R /mnt
cryptsetup close root
reboot
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
