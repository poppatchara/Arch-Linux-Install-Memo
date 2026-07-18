# Arch Linux Installation Guide 🐧

Personal notes for rebuilding my daily Arch install: UEFI, single NVMe, Btrfs root.
Written for SSH remote install — every code block is copy-paste ready.
Not the best way. Just the way I like.

---

## Decision Matrix

Choose ONE per row. Each decision maps to a section.

| # | Decision | A | B | C | § |
|---|----------|---|---|---|---|
| 1 | **Kernel** | `linux-zen` | `linux-cachyos` | `linux` / `linux-lts` | §3 |
| 2 | **Repos** | Vanilla Arch | CachyOS repos | | §9 |
| 3 | **Desktop** | KDE Plasma | Niri + Noctalia | | §7 |
| 4 | **Bootloader** | GRUB | Limine | | §1,§2,§5 |

> **Kernel & repos are independent.** You can use `linux-cachyos` without CachyOS repos, or CachyOS repos with `linux-zen`.

### Recommended Combos

| Combo | Kernel | Repos | Desktop | Bootloader |
|-------|--------|-------|---------|------------|
| ⭐ **KDE Daily** | linux-zen | Vanilla | KDE Plasma | GRUB |
| 🏔️ **Niri Lean** | linux-zen | Vanilla | Niri+Noctalia | GRUB |
| 🚀 **CachyOS KDE** | linux-cachyos | CachyOS | KDE Plasma | Limine |

---

## §0 — Live ISO Prep

Boot Arch ISO. Set root password for SSH:

```bash
passwd
ip a | grep 'inet '  # note the IP
```

SSH in from your client machine:

```bash
ssh root@<IP>
```

### 0.1 Pacman Config

```bash
conf=/etc/pacman.conf
perl -pi -e '
  s/^(ParallelDownloads\s*=\s*)5/${1}15/;
  s/^#Color/Color/;
' "$conf"
perl -0777 -pi -e '
s/^#\[(multilib)\]\n#(Include\s*=\s*\/etc\/pacman\.d\/mirrorlist)(\n)/[\1]\n\2\3/mg
' "$conf"
pacman -Syy
```

### 0.2 Mirror List

Move your country's mirrors before Worldwide:

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
    for (i = cs + 1; i <= n; ++i) { if (lines[i] ~ /^##[[:space:]]+/) { ce = i; break } }
    m = 0
    for (i = cs; i < ce; ++i) { c[++m] = lines[i] }
    while (m > 0 && c[m] ~ /^[[:space:]]*$/) { --m }
    for (i = 1; i < ws; ++i) { if (i < cs || i >= ce) print lines[i] }
    for (i = 1; i <= m; ++i) print c[i]
    print ""
    for (i = ws; i <= n; ++i) { if (i < cs || i >= ce) print lines[i] }
  }
' "$mirrorfile" > "$tmpfile" || { echo "mirrorlist reorder failed; restoring backup." >&2
  cp "${mirrorfile}.bak" "$mirrorfile"; rm -f "$tmpfile"; exit 1; }
cp "$tmpfile" "$mirrorfile"
rm -f "$tmpfile"
pacman -Syy
```

---

## §1 — Partition & Format

> **Decision: GRUB or Limine?** GRUB reads Btrfs → `/boot` can be a Btrfs subvolume (included in snapshots). Limine only reads FAT → kernel/initramfs must live on the ESP.

---

### ▸ GRUB Layout

| Partition | Size | Type | Mount |
|-----------|------|------|-------|
| p1 | 2–4G | EFI System | `/boot/EFI` |
| p2 | RAM-sized | Linux swap | swap |
| p3 | Remainder | Btrfs root | `/` |

#### Step 1 — Create Partitions with cfdisk

```bash
cfdisk /dev/nvme0n1
```

1. If prompted, select **GPT** label type
2. **Create p1:** `[New]` → `2G` (or `4G`) → `[Type]` → `EFI System`
3. **Create p2:** `↓` to free space → `[New]` → (your RAM size, e.g. `32G`) → `[Type]` → `Linux swap`
4. **Create p3:** `↓` to free space → `[New]` → (accept default = remainder) → `[Type]` → `Linux filesystem`
5. `[Write]` → type `yes` → `[Quit]`

#### Step 2 — Format

```bash
mkfs.fat -F32 -n EFI /dev/nvme0n1p1
mkswap /dev/nvme0n1p2
mkfs.btrfs -f -L Arch /dev/nvme0n1p3
```

#### Step 3 — Capture UUIDs

```bash
esp_part="$(lsblk -no PATH,PARTTYPE | while read -r part type; do
  case "$type" in c12a7328-f81f-11d2-ba4b-00a0c93ec93b) echo "$part"; break ;; esac
done)"
swap_part="$(blkid -t TYPE=swap -o device | head -1)"
esp_uuid="$(blkid -s UUID -o value "$esp_part")"
swap_uuid="$(blkid -s UUID -o value "$swap_part")"
root_uuid="$(blkid -s UUID -o value /dev/nvme0n1p3)"
```

---

### ▸ Limine Layout

| Partition | Size | Type | Mount |
|-----------|------|------|-------|
| p1 | 2–4G | EFI System | `/boot` |
| p2 | Remainder | Btrfs root | `/` |
| p3 | RAM-sized | Linux swap | swap |

#### Step 1 — Create Partitions with cfdisk

```bash
cfdisk /dev/nvme0n1
```

1. If prompted, select **GPT** label type
2. **Create p1:** `[New]` → `2G` (or `4G`) → `[Type]` → `EFI System`
3. **Create p2:** `↓` to free space → `[New]` → (accept default = remainder) → `[Type]` → `Linux filesystem`
4. **Create p3:** `↓` to free space → `[New]` → (your RAM size, e.g. `32G`) → `[Type]` → `Linux swap`
5. `[Write]` → type `yes` → `[Quit]`

#### Step 2 — Format

```bash
mkfs.fat -F32 -n EFI /dev/nvme0n1p1
mkfs.btrfs -f -L Arch /dev/nvme0n1p2
mkswap /dev/nvme0n1p3
```

#### Step 3 — Capture UUIDs

```bash
esp_uuid="$(blkid -s UUID -o value /dev/nvme0n1p1)"
root_uuid="$(blkid -s UUID -o value /dev/nvme0n1p2)"
swap_uuid="$(blkid -s UUID -o value /dev/nvme0n1p3)"
```

---

## §2 — Btrfs Subvolumes & Mounts

> **Decision:** Subvolume layout depends on bootloader. GRUB adds `@boot` for snapshot-able kernels. Limine keeps kernels on FAT ESP.

### 2.1 Common Subvolumes

```bash
mount UUID="${root_uuid}" /mnt

btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@var_log
btrfs subvolume create /mnt/@var_cache
btrfs subvolume create /mnt/@root
```

> Optional: add `@home_cache`, `@home_downloads`, `@home_git` subvolumes if you want to exclude them from snapshots.

### ▸ 2.2 GRUB Mounts

```bash
btrfs subvolume create /mnt/@boot

umount -R /mnt
mount -o compress=zstd:1,noatime,subvol=@ UUID="${root_uuid}" /mnt
mount --mkdir -o compress=zstd:1,noatime,subvol=@home     UUID="${root_uuid}" /mnt/home
mount --mkdir -o compress=zstd:1,noatime,subvol=@var_log  UUID="${root_uuid}" /mnt/var/log
mount --mkdir -o compress=zstd:1,noatime,subvol=@var_cache UUID="${root_uuid}" /mnt/var/cache
mount --mkdir -o compress=zstd:1,noatime,subvol=@root     UUID="${root_uuid}" /mnt/root
mount --mkdir -o compress=zstd:1,noatime,subvol=@boot     UUID="${root_uuid}" /mnt/boot
mount --mkdir UUID="${esp_uuid}" /mnt/boot/EFI
swapon UUID="${swap_uuid}"
```

### ▸ 2.3 Limine Mounts

```bash
btrfs subvolume create /mnt/@srv

umount -R /mnt
mount -o compress=zstd:1,noatime,subvol=@ UUID="${root_uuid}" /mnt
mount --mkdir -o compress=zstd:1,noatime,subvol=@home     UUID="${root_uuid}" /mnt/home
mount --mkdir -o compress=zstd:1,noatime,subvol=@var_log  UUID="${root_uuid}" /mnt/var/log
mount --mkdir -o compress=zstd:1,noatime,subvol=@var_cache UUID="${root_uuid}" /mnt/var/cache
mount --mkdir -o compress=zstd:1,noatime,subvol=@root     UUID="${root_uuid}" /mnt/root
mount --mkdir -o compress=zstd:1,noatime,subvol=@srv      UUID="${root_uuid}" /mnt/srv
mount --mkdir UUID="${esp_uuid}" /mnt/boot
swapon UUID="${swap_uuid}"
```

### 2.4 fstab

```bash
mkdir -p /mnt/etc
genfstab -U /mnt > /mnt/etc/fstab
cat /mnt/etc/fstab  # sanity check
```

---

## §3 — Base Install

> **Decision: Kernel.** Pick one. All are in official repos:
> - `linux-zen` — tuned for desktop responsiveness
> - `linux-cachyos` — CachyOS optimized (add CachyOS repos in §9, or use `pacman -U` with downloaded package)
> - `linux` — vanilla stable
> - `linux-lts` — long-term support

> **Decision: Repos.** Vanilla (nothing extra) or CachyOS (add in §9 post-install). Kernel choice is independent of repo choice.

### 3.0 CPU Detection

```bash
cpu=intel
lscpu | grep -qi amd && cpu=amd
```

### 3.1 Set Kernel Variable

```bash
# Pick ONE:
KERNEL=linux-zen
# KERNEL=linux-cachyos
# KERNEL=linux
# KERNEL=linux-lts
```

### 3.2 vconsole

```bash
cat <<'EOF' > /mnt/etc/vconsole.conf
KEYMAP=us
FONT=ter-124n
EOF
```

### 3.3 pacstrap

```bash
pacstrap -K /mnt \
  base base-devel \
  "${KERNEL}" "${KERNEL}-headers" linux-firmware "${cpu}-ucode" \
  efibootmgr btrfs-progs dosfstools e2fsprogs exfatprogs \
  networkmanager openssh \
  nvim git sudo man curl \
  zsh zsh-completions zsh-autosuggestions bash-completion tmux \
  pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber reflector

cp /etc/pacman.conf /mnt/etc/pacman.conf
cp /etc/pacman.d/mirrorlist /mnt/etc/pacman.d/mirrorlist
```

### 3.4 Enter chroot

```bash
arch-chroot /mnt
```

---

## §4 — Chroot Configuration

### 4.1 Safety Check

```bash
[ "$(findmnt -n -o FSTYPE /)" = "btrfs" ] || { echo "ERROR: Not in chroot"; exit 1; }
```

### 4.2 Locale & Timezone

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

### 4.3 Hostname

```bash
host_name="arch"
echo "${host_name}" > /etc/hostname
cat <<EOF > /etc/hosts
127.0.0.1   localhost
::1         localhost
127.0.1.1   ${host_name}.localdomain ${host_name}
EOF
```

### 4.4 Users & Sudo

```bash
echo "Set root password:"
passwd

user=pop
groupadd -f docker
useradd -m -G wheel,storage,power,audio,video,docker -s /bin/bash $user
echo "Set password for ${user}:"
passwd $user

# Uncomment %wheel ALL=(ALL) ALL
EDITOR=nvim visudo
```

### 4.5 mkinitcpio

```bash
perl -pi -e '
  s/^MODULES=\(\)/MODULES=(btrfs)/;
  s/^BINARIES=\(\)/BINARIES=(\/usr\/bin\/btrfs)/;
  s/^HOOKS=\(.*\)$/HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block encrypt filesystems resume fsck)/;
' /etc/mkinitcpio.conf

mkinitcpio -P
```

---

## §5 — Bootloader

> **Decision: GRUB or Limine?**

### ▸ 5.1 GRUB

```bash
pacman -S --noconfirm --needed grub
grub-install --target=x86_64-efi --efi-directory=/boot/EFI --bootloader-id=GRUB
```

Configure cmdline:

```bash
ucode_img="intel"
lscpu | grep -qi amd && ucode_img="amd"

swap_part="$(blkid -t TYPE=swap -o device | head -1)"
swap_uuid="$(blkid -s UUID -o value "$swap_part")"

sed -i '/^GRUB_CMDLINE_LINUX_DEFAULT=/d' /etc/default/grub
sed -i "/^GRUB_CMDLINE_LINUX=/a GRUB_CMDLINE_LINUX_DEFAULT=\"loglevel=3 resume=UUID=${swap_uuid} zswap.enabled=1 zswap.compressor=lz4 zswap.max_pool_percent=50 zswap.zpool=zsmalloc ${ucode_img}_iommu=on iommu=pt\"" /etc/default/grub
```

Add zswap modules to initramfs:

```bash
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
mkinitcpio -P
```

Generate config:

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

> **Dual-boot (optional):** `pacman -S --needed os-prober && echo 'GRUB_DISABLE_OS_PROBER=false' >> /etc/default/grub && grub-mkconfig -o /boot/grub/grub.cfg`

### ▸ 5.2 Limine

```bash
pacman -S --noconfirm --needed limine
mkdir -p /boot/EFI/limine /boot/limine
cp -v /usr/share/limine/*.EFI /boot/EFI/limine/

efibootmgr --create --disk /dev/nvme0n1 --part 1 \
  --label "Limine Bootloader" \
  --loader '\EFI\limine\BOOTX64.EFI' \
  --unicode
```

Copy kernel artifacts to ESP and generate config:

```bash
root_uuid="$(blkid -s UUID -o value /dev/nvme0n1p2)"
swap_uuid="$(blkid -s UUID -o value /dev/nvme0n1p3)"

ucode_img="intel"
lscpu | grep -qi amd && ucode_img="amd"

cp -v /boot/vmlinuz-* /boot/limine/
cp -v /boot/initramfs-*.img /boot/limine/
cp -v "/boot/${ucode_img}-ucode.img" /boot/limine/

cat <<EOF | tee /boot/limine/limine.conf >/dev/null
TIMEOUT=3
DEFAULT_ENTRY=Arch Linux

/Arch Linux
    PROTOCOL: linux
    KERNEL_PATH: boot():/limine/vmlinuz-${KERNEL}
    MODULE_PATH: boot():/limine/${ucode_img}-ucode.img
    MODULE_PATH: boot():/limine/initramfs-${KERNEL}.img
    CMDLINE: loglevel=3 root=UUID=${root_uuid} rootflags=subvol=@ rootfstype=btrfs rw resume=UUID=${swap_uuid} zswap.enabled=1 zswap.compressor=lz4 zswap.max_pool_percent=50 zswap.zpool=zsmalloc ${ucode_img}_iommu=on iommu=pt

/Arch Linux (fallback)
    PROTOCOL: linux
    KERNEL_PATH: boot():/limine/vmlinuz-${KERNEL}
    MODULE_PATH: boot():/limine/initramfs-${KERNEL}-fallback.img
    CMDLINE: loglevel=3 root=UUID=${root_uuid} rootflags=subvol=@ rootfstype=btrfs rw
EOF
```

Pacman hook for auto-copy on Limine updates:

```bash
mkdir -p /etc/pacman.d/hooks
tee /etc/pacman.d/hooks/99-limine.hook >/dev/null <<'EOF'
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

Add zswap modules to initramfs (same as GRUB):

```bash
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
mkinitcpio -P

# Re-copy updated initramfs
cp -v /boot/initramfs-*.img /boot/limine/
```

---

## §6 — Services & QoL

### 6.1 Extra Packages

```bash
pacman -Syu --noconfirm --needed \
  util-linux inetutils usbutils rsync htop bat zip unzip p7zip \
  avahi nss-mdns \
  alsa-utils sof-firmware easyeffects \
  bluez bluez-utils \
  acpi acpid \
  xdg-user-dirs
```

### 6.2 Enable Services

```bash
systemctl enable NetworkManager
systemctl enable bluetooth
systemctl enable reflector.timer
systemctl enable sshd
systemctl enable fstrim.timer

# Optional (uncomment if needed):
# systemctl enable iwd
# systemctl enable cups
# systemctl enable avahi-daemon
# systemctl enable acpid
```

---

## §7 — Desktop Stack

> **Decision: KDE Plasma (all-in-one) or Compositor + Shell?**

### ▸ 7.1 KDE Plasma

```bash
# Core login + Wayland
pacman -S --noconfirm --needed \
  plasma-login-manager \
  xdg-desktop-portal xdg-desktop-portal-kde \
  qt6-wayland xorg-xwayland

# Plasma desktop (pulls plasma-workspace, kwin, systemsettings)
pacman -S --noconfirm --needed \
  plasma-desktop \
  plasma-nm plasma-pa kscreen \
  kde-gtk-config breeze-gtk

# Optional Plasma extras
pacman -S --noconfirm --needed \
  bluedevil power-profiles-daemon \
  kdeplasma-addons plasma-systemmonitor \
  plasma-browser-integration discover \
  krdp print-manager

systemctl enable plasmalogin
systemctl enable power-profiles-daemon
```

**PAM config (MANDATORY):**

```bash
tee /etc/pam.d/plasmalogin <<'EOF'
#%PAM-1.0
auth       sufficient   pam_succeed_if.so user ingroup nopasswdlogin
auth       include      system-login
account    include      system-login
session    include      system-login
session    optional     pam_kwallet5.so auto_start kwalletd=/usr/bin/ksecretd
password   include      system-login
EOF
```

**Desktop Apps:**

```bash
pacman -S --noconfirm --needed \
  dolphin dolphin-plugins konsole kate okular gwenview spectacle ark \
  gparted kio-extras ffmpegthumbs kdegraphics-thumbnailers \
  filelight kcalc btop fastfetch
```

**KWallet (optional):**

```bash
pacman -S --noconfirm --needed kwallet kwalletmanager kwallet-pam
```

### ▸ 7.2 Compositor: Niri

Niri is in official repos. Noctalia shell (AUR) is installed post-reboot in §9.

```bash
pacman -S --noconfirm --needed niri
```

**Session file (so greetd/noctalia-greeter can launch Niri):**

```bash
tee /usr/local/bin/niri-session-wrapper << 'EOF'
#!/usr/bin/env bash
export XDG_CURRENT_DESKTOP=Noctalia
export XDG_SESSION_DESKTOP=noctalia
exec niri-session
EOF
chmod +x /usr/local/bin/niri-session-wrapper

tee /usr/share/wayland-sessions/niri-noctalia.desktop <<'EOF'
[Desktop Entry]
Name=Niri + Noctalia v5
Comment=Session with Niri compositor and Noctalia v5 shell
Exec=/usr/local/bin/niri-session-wrapper
Type=Application
DesktopNames=Noctalia
EOF
```

**Base Wayland + KDE integration packages:**

```bash
pacman -S --noconfirm --needed \
  xdg-desktop-portal xdg-desktop-portal-kde \
  qt6-wayland xorg-xwayland xwayland-satellite \
  xdg-utils shared-mime-info kde-cli-tools \
  accountsservice qt5-wayland \
  gvfs-mtp gvfs-gphoto2 udiskie \
  plasma-integration kded qt6ct-kde \
  ghostty libcanberra
```

**Desktop Apps (same as KDE, minus konsole — use ghostty):**

```bash
pacman -S --noconfirm --needed \
  dolphin dolphin-plugins kate okular gwenview spectacle ark \
  gparted kio-extras ffmpegthumbs kdegraphics-thumbnailers \
  filelight kcalc btop fastfetch capitaine-cursors
```

> **KDE background services:** Add `spawn-at-startup "kded6"` to Niri config in §7.3.

### ▸ 7.3 Shell: Noctalia v5

Install post-reboot (AUR, needs yay from §9):

```bash
# After reboot + yay installed (§9.1):
yay -S --noconfirm --needed noctalia-git noctalia-greeter
sudo pacman -S --noconfirm --needed greetd
```

**greetd config:**

```bash
sudo tee /etc/greetd/config.toml <<'EOF'
[terminal]
vt = 1

[default_session]
command = "noctalia-greeter"
user = "greetd"
EOF

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
EOF

sudo systemctl enable greetd
```

**Niri config** (`~/.config/niri/config.kdl`):

```kdl
input {
    keyboard {
        xkb { layout "us,th" options "grp:lalt_lshift_toggle" }
        numlock
    }
    touchpad { tap natural-scroll }
    focus-follows-mouse
    workspace-auto-back-and-forth
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

// Wallpaper — stationary (visible always, doesn't scroll)
layer-rule {
    match namespace="^noctalia-wallpaper"
    place-within-backdrop true
}
layout {
    gaps 16
    background-color "transparent"
    center-focused-column "never"
    preset-column-widths { proportion 0.33333 proportion 0.5 proportion 0.66667 }
}
overview { workspace-shadow { off } }

// Blur
window-rule { background-effect { blur true xray false } }
layer-rule {
    match namespace="^noctalia-(bar-[^\"]+|notification|dock|panel|attached-panel|osd)$"
    background-effect { xray false }
}
blur { passes 2 offset 3.0 noise 0.03 saturation 1.0 }

binds {
    // Noctalia IPC
    Mod+Space   { spawn-sh "noctalia msg panel-toggle launcher"; }
    Mod+S       { spawn-sh "noctalia msg panel-toggle control-center"; }
    Mod+Comma   { spawn-sh "noctalia msg settings-toggle"; }
    Mod+Tab     { toggle-overview; }
    Mod+Shift+S { spawn "spectacle -r"; }

    // Audio & brightness
    XF86AudioRaiseVolume  { spawn-sh "noctalia msg volume-up; canberra-gtk-play -i audio-volume-change -d 'volume feedback'"; }
    XF86AudioLowerVolume  { spawn-sh "noctalia msg volume-down; canberra-gtk-play -i audio-volume-change -d 'volume feedback'"; }
    XF86AudioMute         { spawn-sh "noctalia msg volume-mute; canberra-gtk-play -i audio-volume-change -d 'volume feedback'"; }
    XF86MonBrightnessUp   { spawn-sh "noctalia msg brightness-up"; }
    XF86MonBrightnessDown { spawn-sh "noctalia msg brightness-down"; }

    // Apps
    Mod+Return  { spawn "ghostty"; }
    Mod+E       { spawn "dolphin"; }
    Mod+B       { spawn "firefox"; }
}

// Screenshots
binds {
    Print { spawn "spectacle -f"; }
    Ctrl+Print { spawn "spectacle -a"; }
    Alt+Print { spawn "spectacle -w"; }
}

environment {
    ELECTRON_OZONE_PLATFORM_HINT "auto"
    QT_QPA_PLATFORM "wayland"
    QT_QPA_PLATFORMTHEME "qt6ct"
    QT_WAYLAND_DISABLE_WINDOWDECORATION "1"
    XDG_CURRENT_DESKTOP "niri"
    XDG_SESSION_TYPE "wayland"
}

cursor { xcursor-theme "capitaine-cursors" xcursor-size 24 }

hotkey-overlay { skip-at-startup }
```

> **Full config reference:** See companion guide `Niri_Noctalia_v5.md` for animations, gaming rules, modular config structure, greeter reference, and troubleshooting.

---

## §8 — Reboot

```bash
exit          # exit chroot
umount -R /mnt
swapoff -a
reboot
```

Remove installation media. Log in as your user.

---

## §9 — Post-Install

### 9.1 XDG User Dirs

```bash
xdg-user-dirs-update
```

### 9.2 YAY (AUR Helper)

```bash
sudo pacman -S --noconfirm --needed git base-devel
git clone https://aur.archlinux.org/yay.git && cd yay && makepkg -si
cd .. && rm -rf yay
```

> **AUR Security:** Always review PKGBUILDs. Look for suspicious URLs, obfuscated code, or curl-pipe-bash patterns. Packages marked 🔒 need manual review.

### 9.3 CachyOS Repos & Kernel (optional)

> Skip if using Vanilla Arch repos.

```bash
# Backup
sudo cp -a /etc/pacman.conf /etc/pacman.conf.pre-cachy
pacman -Qqe > ~/pkglist.pre-cachy.txt

# Add CachyOS repos
cd ~
curl https://mirror.cachyos.org/cachyos-repo.tar.xz -o cachyos-repo.tar.xz
tar xvf cachyos-repo.tar.xz && cd cachyos-repo
sudo ./cachyos-repo.sh

# Install CachyOS kernels
sudo pacman -Syu
sudo pacman -S linux-cachyos linux-cachyos-headers linux-cachyos-eevdf linux-cachyos-eevdf-headers

# Optional: rebuild all packages with CachyOS optimizations
# sudo pacman -Qqn | sudo pacman -S -

# Optional CachyOS extras
sudo pacman -S cachyos-settings cachyos-gaming-meta cachyos-hello
```

### 9.4 GPU Driver

```bash
# Detect GPU
gpu_vendor=""
if lspci | grep -qi nvidia; then gpu_vendor="nvidia"
elif lspci | grep -qiE "intel.*(graphic|display|UHD|Iris|Arc)"; then gpu_vendor="intel"
elif lspci | grep -qiE "amd.*(graphic|radeon|advanced)"; then gpu_vendor="amd"
fi
echo "Detected GPU: ${gpu_vendor:-unknown}"
```

**NVIDIA (if detected):**

```bash
if [ "$gpu_vendor" = "nvidia" ]; then
  sudo pacman -S --noconfirm --needed \
    nvidia-open-dkms nvidia-utils lib32-nvidia-utils \
    nvidia-settings libxnvctrl \
    ocl-icd opencl-nvidia lib32-opencl-nvidia clinfo cuda

  # Gaming extras
  sudo pacman -S --noconfirm --needed \
    libva-utils vdpauinfo vulkan-tools \
    libva-nvidia-driver dxvk vkd3d shaderc spirv-tools

  # DRM kernel params (GRUB)
  if grep -q 'GRUB_CMDLINE_LINUX_DEFAULT' /etc/default/grub 2>/dev/null; then
    sudo sed -i '/^GRUB_CMDLINE_LINUX_DEFAULT=/s/"$/ nvidia-drm.modeset=1 nvidia-drm.fbdev=1"/' /etc/default/grub
    sudo grub-mkconfig -o /boot/grub/grub.cfg
  fi

  # NVIDIA modules + remove kms hook
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

  # Pacman hook for auto-rebuild
  sudo mkdir -p /etc/pacman.d/hooks
  sudo tee /etc/pacman.d/hooks/nvidia.hook >/dev/null <<'HOOK'
[Trigger]
Operation = Install
Operation = Upgrade
Operation = Remove
Type = Package
Target = nvidia-open-dkms
Target = linux
Target = linux-zen
Target = linux-cachyos

[Action]
Description = Update Nvidia module in initramfs
Depends = mkinitcpio
When = PostTransaction
NeedsTargets
Exec = /bin/sh -c 'while read -r trg; do case $trg in linux*) exit 0; esac; done; /usr/bin/mkinitcpio -P'
HOOK
fi
```

### 9.5 Snapper

```bash
# Install
sudo pacman -S --noconfirm --needed snapper btrfs-assistant
yay -S --noconfirm --needed grub-btrfs snap-pac snap-pac-grub snapper-gui-git  # 🔒

# Create configs
sudo snapper -c root create-config /
sudo snapper -c home create-config /home   # skip if you don't want /home snapshots

# GRUB only:
sudo snapper -c boot create-config /boot
sudo systemctl enable --now grub-btrfsd
```

**Retention settings:**

```bash
# root — hourly for quick undo
sudo snapper -c root set-config TIMELINE_CREATE=yes
sudo snapper -c root set-config TIMELINE_LIMIT_HOURLY=5 TIMELINE_LIMIT_DAILY=7 TIMELINE_LIMIT_WEEKLY=4
sudo snapper -c root set-config NUMBER_LIMIT=15 NUMBER_LIMIT_IMPORTANT=5

# home — light retention
sudo snapper -c home set-config TIMELINE_CREATE=yes
sudo snapper -c home set-config TIMELINE_LIMIT_HOURLY=0 TIMELINE_LIMIT_DAILY=3 TIMELINE_LIMIT_WEEKLY=2
sudo snapper -c home set-config NUMBER_LIMIT=10 NUMBER_LIMIT_IMPORTANT=3

# boot (GRUB) — snap-pac covers kernel updates, no timeline
sudo snapper -c boot set-config TIMELINE_CREATE=no 2>/dev/null
sudo snapper -c boot set-config NUMBER_LIMIT=5 NUMBER_LIMIT_IMPORTANT=3 2>/dev/null

# Enable timers
sudo systemctl enable --now snapper-timeline.timer snapper-cleanup.timer
```

### 9.6 SSH Hardening

```bash
sudo tee -a /etc/ssh/sshd_config <<'EOF'
PermitRootLogin no
PasswordAuthentication no
PermitEmptyPasswords no
KbdInteractiveAuthentication no
MaxAuthTries 3
LoginGraceTime 60
StrictModes yes
HostbasedAuthentication no
IgnoreRhosts yes
ClientAliveInterval 300
ClientAliveCountMax 2
EOF

sudo systemctl restart sshd
```

> ⚠️ **Test before disconnecting!** Open a new terminal and verify key-based login works first.

### 9.7 Extra Packages & Fonts

```bash
# Core CLI + media
sudo pacman -S --noconfirm --needed imagemagick vlc gimp obs-studio

# Filesystem / network
sudo pacman -S --noconfirm --needed gvfs gvfs-smb brightnessctl

# Node.js
sudo pacman -S --noconfirm --needed nodejs npm bun

# Browsers + office
sudo pacman -S --noconfirm --needed firefox chromium libreoffice-fresh filezilla
yay -S --noconfirm --needed brave-bin visual-studio-code-bin  # 🔒

# Gaming
sudo pacman -S --noconfirm --needed gamemode lib32-gamemode steam lutris mangohud lib32-mangohud goverlay
yay -S --noconfirm --needed proton-ge-custom-bin  # 🔒
sudo usermod -aG gamemode $USER

# Fonts
sudo pacman -S --noconfirm --needed \
  noto-fonts noto-fonts-emoji \
  ttf-dejavu ttf-ubuntu-font-family \
  terminus-font nerd-fonts ttf-ms-fonts

# Flatpak (optional)
sudo pacman -S --noconfirm --needed flatpak
```

### 9.8 pyenv

```bash
sudo pacman -S --noconfirm --needed openssl zlib xz tk readline sqlite libffi bzip2
git clone https://github.com/pyenv/pyenv.git ~/.pyenv

shell_name="$(basename "${SHELL:-bash}")"
conf="$HOME/.bashrc"
[ "$shell_name" = "zsh" ] && conf="$HOME/.zshrc"
touch "$conf"

if ! grep -qF 'PYENV_ROOT' "$conf"; then
  cat <<'EOF' >> "$conf"
# pyenv
export PYENV_ROOT="$HOME/.pyenv"
export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init -)"
EOF
fi

exec "$SHELL"
pyenv install 3.13.2
pyenv global 3.13.2
```

### 9.9 SPDIF Audio Fix (optional)

If your SPDIF DAC drops the first 1–3 seconds of audio:

```bash
# Disable codec autosuspend
sudo tee /etc/modprobe.d/alsa-no-powersave.conf >/dev/null <<'EOF'
options snd_hda_intel power_save=0 power_save_controller=N
EOF

# Stop WirePlumber from suspending sinks
mkdir -p ~/.config/wireplumber/main.lua.d
cat <<'EOF' > ~/.config/wireplumber/main.lua.d/51-spdif-nosuspend.lua
alsa_monitor.rules = alsa_monitor.rules or {}
table.insert(alsa_monitor.rules, {
  matches = { { { "node.name", "matches", "alsa_output.*" } } },
  apply_properties = { ["session.suspend-timeout-seconds"] = 0 }
})
EOF
systemctl --user restart wireplumber
```

### 9.10 KWallet (KDE only)

Disable if you don't use it:

```bash
mkdir -p ~/.config
cat <<'EOF' >> ~/.config/kwalletrc
[Wallet]
Enabled=false
EOF
```

### 9.11 Cache Cleanup

```bash
sudo pacman -Scc
```

---

## Credits

- [Arch Wiki](https://wiki.archlinux.org/)
- [Limine](https://github.com/limine-bootloader/limine)
- [CachyOS](https://cachyos.org/)
- [Noctalia v5 docs](https://docs.noctalia.dev/v5/)
- [Niri Arch Wiki](https://wiki.archlinux.org/title/Niri)
- Various gists & community notes linked inline

*Last updated: July 2026*
