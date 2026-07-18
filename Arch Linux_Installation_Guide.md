# Arch Linux Installation Guide 🐧

Personal notes for rebuilding my daily Arch install: UEFI, single NVMe, Btrfs root.
Written for SSH remote install — every code block is copy-paste ready.
Not the best way. Just the way I like.

---

## Contents

- [Decision Matrix](#decision-matrix)
- [§0 — Live ISO Prep](#0--live-iso-prep)
  - [0.0 SSH Setup](#00-ssh-setup)
  - [0.1 Pacman Config](#01-pacman-config)
  - [0.2 Mirror List](#02-mirror-list)
- [§1 — Partition & Format](#1--partition--format)
  - [▸ GRUB](#-grub)
  - [▸ Limine](#-limine)
- [§2 — Btrfs Subvolumes & Mounts](#2--btrfs-subvolumes--mounts)
  - [2.1 Common Subvolumes](#21-common-subvolumes)
  - [▸ 2.2 GRUB Mounts](#-22-grub-mounts)
  - [▸ 2.3 Limine Mounts](#-23-limine-mounts)
  - [2.4 fstab](#24-fstab)
- [§3 — Base Install](#3--base-install)
  - [3.0 CPU Detection](#30-cpu-detection)
  - [3.1 Select Kernels](#31-select-kernels)
  - [3.2 vconsole](#32-vconsole)
  - [3.3 pacstrap](#33-pacstrap)
  - [3.4 Enter chroot](#34-enter-chroot)
- [CachyOS Repos](#cachyos-repos-optional)
- [§4 — Chroot Configuration](#4--chroot-configuration)
  - [4.1 Safety Check](#41-safety-check)
  - [4.2 Locale & Timezone](#42-locale--timezone)
  - [4.3 Hostname](#43-hostname)
  - [4.4 Users & Sudo](#44-users--sudo)
  - [4.5 mkinitcpio](#45-mkinitcpio)
- [§5 — Bootloader](#5--bootloader)
  - [▸ GRUB](#-grub-1)
  - [▸ Limine](#-limine-1)
- [§6 — Services & QoL](#6--services--qol)
  - [6.1 Extra Packages](#61-extra-packages)
  - [6.2 Enable Services](#62-enable-services)
- [§7 — Desktop Stack](#7--desktop-stack)
  - [▸ KDE Plasma](#-kde-plasma)
  - [▸ Compositor: Niri](#-compositor-niri)
  - [▸ Shell: Noctalia v5](#-shell-noctalia-v5)
- [§8 — Reboot](#8--reboot)
- [§9 — Post-Install](#9--post-install)
  - [9.1 XDG User Dirs](#91-xdg-user-dirs)
  - [9.2 YAY](#92-yay-aur-helper)
  - [9.3 CachyOS Extras](#93-cachyos-extras-optional)
  - [9.4 GPU Driver](#94-gpu-driver)
  - [9.5 Snapper](#95-snapper)
  - [9.6 SSH Hardening](#96-ssh-hardening)
  - [9.7 Firewall](#97-firewall)
  - [9.8 AppArmor](#98-apparmor-optional)
  - [9.9 Extra Packages & Fonts](#99-extra-packages--fonts)
  - [9.10 pyenv](#910-pyenv)
  - [9.11 SPDIF Audio Fix](#911-spdif-audio-fix-optional)
  - [9.12 Cache Cleanup](#912-cache-cleanup)
- [Credits](#credits)

---

## Decision Matrix

Choose ONE per row (multiple kernels OK). Each choice maps to the section where it takes effect.

| # | Decision | A | B | C | § |
|---|----------|---|---|---|---|
| 1 | **Kernel** | `linux-zen` | `linux-cachyos` | `linux` / `linux-lts` | §3 |
| 2 | **Repos** | Vanilla Arch | CachyOS repos | | §9 |
| 3 | **Desktop** | KDE Plasma | Niri + Noctalia | | §7 |
| 4 | **Bootloader** | GRUB | Limine | | §1,§2,§5 |

> **Kernel & repos are independent.** You can use `linux-cachyos` without CachyOS repos, or CachyOS repos with `linux-zen`. `linux-cachyos` is in the official `[extra]` repo — no third-party repo needed.

### Recommended Combos

These are the three configurations this guide has been battle-tested with:

| Combo | Kernel | Repos | Desktop | Bootloader |
|-------|--------|-------|---------|------------|
| ⭐ **KDE Daily** | linux-zen | Vanilla | KDE Plasma | GRUB |
| 🏔️ **Niri Lean** | linux-zen | Vanilla | Niri+Noctalia | GRUB |
| 🚀 **CachyOS KDE** | linux-cachyos | CachyOS | KDE Plasma | Limine |

---

## §0 — Live ISO Prep

The Arch ISO boots you into a minimal live environment. We'll configure it for fast package downloads, then SSH in from a client machine so we can copy-paste commands comfortably.

### 0.0 SSH Setup

The live ISO runs an SSH server — you just need to set a root password and find the IP:

```bash
passwd
ip a | grep 'inet '  # note the IP address
```

Now from your client machine (the one with a real keyboard and a browser for reading this guide):

```bash
ssh root@<IP>
```

Every command from here on runs over SSH. Type carefully — there's no GUI to fall back on.

### 0.1 Pacman Config

Speed up downloads and enable the `[multilib]` repo (needed for 32-bit libraries like Steam):

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

Arch ships with a global mirror list. Moving your country's mirrors to the top means faster download speeds:

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

> ⚠️ **Pick ONE bootloader below.** These are alternatives — do not install both.

The bootloader determines the partition layout. The key difference:

- **GRUB** can read Btrfs natively. This means `/boot` (kernels + initramfs) can live on a Btrfs subvolume, getting included in snapshots and rollbacks.
- **Limine** only reads FAT. Kernels and initramfs must be copied to the EFI System Partition (FAT32), which is outside snapshot coverage.

This affects where swap goes too — we want swap at the end for Limine (easy to resize away), and in the middle for GRUB (maximizing contiguous Btrfs space, with `/boot/EFI` at the edge).

---

### ▸ GRUB

ESP mounted at `/boot/EFI`, swap in the middle, Btrfs takes the rest:

| Partition | Size | Type | Mount |
|-----------|------|------|-------|
| p1 | 2–4G | EFI System | `/boot/EFI` |
| p2 | RAM-sized | Linux swap | swap |
| p3 | Remainder | Btrfs root | `/` |

#### Step 1 — Create Partitions with cfdisk

`cfdisk` is a terminal-based partition editor. Use arrow keys, Enter, and Tab to navigate:

```bash
cfdisk /dev/nvme0n1
```

1. If prompted, select **GPT** label type
2. **Create p1:** `[New]` → `2G` (or `4G`) → `[Type]` → `EFI System`
3. **Create p2:** `↓` to free space → `[New]` → (your RAM size, e.g. `32G`) → `[Type]` → `Linux swap`
4. **Create p3:** `↓` to free space → `[New]` → (accept default = remainder) → `[Type]` → `Linux filesystem`
5. `[Write]` → type `yes` → `[Quit]`

#### Step 2 — Format

Each partition gets its filesystem. `mkfs.fat` for the ESP (UEFI firmware requirement), `mkswap` for swap, `mkfs.btrfs` for the root. The `-L` label is cosmetic — helps identify the disk in file managers:

```bash
mkfs.fat -F32 -n EFI /dev/nvme0n1p1
mkswap /dev/nvme0n1p2
mkfs.btrfs -f -L Arch /dev/nvme0n1p3
```

#### Step 3 — Capture UUIDs

UUIDs are stable identifiers — unlike `/dev/nvme0n1pN` which can change if disk topology shifts. We detect by filesystem label (`EFI`) and type (`swap`, `btrfs`) so the guide works on any disk:

```bash
esp_part="$(blkid -L EFI -o device)"
esp_uuid="$(blkid -s UUID -o value "$esp_part")"
swap_part="$(blkid -t TYPE=swap -o device | head -1)"
swap_uuid="$(blkid -s UUID -o value "$swap_part")"
root_part="$(blkid -t TYPE=btrfs -o device | head -1)"
root_uuid="$(blkid -s UUID -o value "$root_part")"
```

---

### ▸ Limine

ESP mounted at `/boot` directly (because Limine reads kernels from FAT), swap at the end:

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

Same label-based detection as GRUB:

```bash
esp_part="$(blkid -L EFI -o device)"
esp_uuid="$(blkid -s UUID -o value "$esp_part")"
swap_part="$(blkid -t TYPE=swap -o device | head -1)"
swap_uuid="$(blkid -s UUID -o value "$swap_part")"
root_part="$(blkid -t TYPE=btrfs -o device | head -1)"
root_uuid="$(blkid -s UUID -o value "$root_part")"
```

---

## §2 — Btrfs Subvolumes & Mounts

> **Decision:** Subvolume layout depends on bootloader.

Btrfs subvolumes are like lightweight partitions inside a single filesystem. They share the same space but can be snapshotted independently. This matters because:

- `@` (root) gets snapshot coverage via Snapper
- `@var_log` and `@var_cache` are isolated — they change constantly but we don't need to snapshot them
- `@var_tmp` for `/var/tmp` — temporary files, excluded from snapshots
- `@home` gets light snapshot coverage (optional)
- `@root` keeps `/root` (the root user's home) separate

The `@` naming convention came from openSUSE's Snapper layout. It's not required, but most tooling expects it.

Mount options:
- `compress=zstd:1` — transparent compression at level 1 (fast). Btrfs compresses data before writing to disk. Level 1 is near-zero CPU overhead.
- `noatime` — don't update file access timestamps. Significantly reduces metadata writes, especially on SSDs.

### 2.1 Common Subvolumes

First, mount the Btrfs root so we can create subvolumes inside it:

```bash
mount UUID="${root_uuid}" /mnt

btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@var_log
btrfs subvolume create /mnt/@var_cache
btrfs subvolume create /mnt/@var_tmp
btrfs subvolume create /mnt/@root
```

> Optional: add `@home_cache`, `@home_downloads`, `@home_git` subvolumes if you want to exclude those directories from snapshots (they tend to be large and change frequently).

### ▸ 2.2 GRUB Mounts

GRUB adds `@boot` — this is the key advantage. Kernel updates get snapshotted automatically by `snap-pac`. After creating it, we unmount everything and re-mount each subvolume with the correct options:

```bash
btrfs subvolume create /mnt/@boot

umount -R /mnt
mount -o compress=zstd:1,noatime,subvol=@ UUID="${root_uuid}" /mnt
mount --mkdir -o compress=zstd:1,noatime,subvol=@home     UUID="${root_uuid}" /mnt/home
mount --mkdir -o compress=zstd:1,noatime,subvol=@var_log  UUID="${root_uuid}" /mnt/var/log
mount --mkdir -o compress=zstd:1,noatime,subvol=@var_cache UUID="${root_uuid}" /mnt/var/cache
mount --mkdir -o compress=zstd:1,noatime,subvol=@var_tmp   UUID="${root_uuid}" /mnt/var/tmp
mount --mkdir -o compress=zstd:1,noatime,subvol=@root     UUID="${root_uuid}" /mnt/root
mount --mkdir -o compress=zstd:1,noatime,subvol=@boot     UUID="${root_uuid}" /mnt/boot
mount --mkdir UUID="${esp_uuid}" /mnt/boot/EFI
swapon UUID="${swap_uuid}"
```

The ESP is mounted at `/boot/EFI` — inside the Btrfs `/boot`. GRUB reads the kernel from Btrfs `/boot`, then chain-loads from the FAT32 ESP at `/boot/EFI`.

### ▸ 2.3 Limine Mounts

Limine can't read Btrfs, so the ESP is mounted directly at `/boot`. No `@boot` subvolume needed — kernel artifacts get copied to FAT32 in §5.2. We add `@srv` instead for server data separation:

```bash
btrfs subvolume create /mnt/@srv

umount -R /mnt
mount -o compress=zstd:1,noatime,subvol=@ UUID="${root_uuid}" /mnt
mount --mkdir -o compress=zstd:1,noatime,subvol=@home     UUID="${root_uuid}" /mnt/home
mount --mkdir -o compress=zstd:1,noatime,subvol=@var_log  UUID="${root_uuid}" /mnt/var/log
mount --mkdir -o compress=zstd:1,noatime,subvol=@var_cache UUID="${root_uuid}" /mnt/var/cache
mount --mkdir -o compress=zstd:1,noatime,subvol=@var_tmp   UUID="${root_uuid}" /mnt/var/tmp
mount --mkdir -o compress=zstd:1,noatime,subvol=@root     UUID="${root_uuid}" /mnt/root
mount --mkdir -o compress=zstd:1,noatime,subvol=@srv      UUID="${root_uuid}" /mnt/srv
mount --mkdir UUID="${esp_uuid}" /mnt/boot
swapon UUID="${swap_uuid}"
```

### 2.4 fstab

`genfstab` generates `/etc/fstab` from the current mount state. Using `-U` writes UUIDs (not device paths). Always glance at the output before moving on — a misconfigured fstab means an unbootable system:

```bash
mkdir -p /mnt/etc
genfstab -U /mnt > /mnt/etc/fstab
cat /mnt/etc/fstab  # sanity check
```

---

## §3 — Base Install

> **Decision: Kernel.**

- `linux-zen` — kernel tuned for desktop/laptop responsiveness (lower latency, different scheduler defaults). My daily driver.
- `linux-cachyos` — CachyOS's default optimized kernel. Installable from `[extra]` without CachyOS repos.
- `linux-cachyos-bore` — CachyOS variant with BORE scheduler. Needs CachyOS repos.
- `linux-cachyos-eevdf` — CachyOS variant with EEVDF scheduler. Needs CachyOS repos.
- `linux` — vanilla stable kernel. Conservative, well-tested.
- `linux-lts` — long-term support. Older but extremely stable. Good fallback.

> **Decision: Repos.** Vanilla Arch (`[core]`, `[extra]`, `[multilib]`) vs adding CachyOS repos (done in §9 post-install). Kernel choice is independent.

### 3.0 CPU Detection

The `*-ucode` package loads CPU microcode updates at boot — critical for security and stability:

```bash
cpu=intel
lscpu | grep -qi amd && cpu=amd
```

### 3.1 Select Kernels

Install at least one. You can install multiple — common combos: `linux-zen` (daily) + `linux-lts` (fallback), or `linux-cachyos` + `linux-zen`.

```bash
# Uncomment the kernels you want:
KERNELS=(
  linux-zen
  # linux-cachyos        # in [extra] — no extra repo needed
  # linux-cachyos-bore    # needs CachyOS repos → auto-added below
  # linux-cachyos-eevdf   # needs CachyOS repos → auto-added below
  # linux
  # linux-lts
)

# Build kernel package list for pacstrap
KERNEL_PKGS=()
NEED_CACHYOS=0
for k in "${KERNELS[@]}"; do
  KERNEL_PKGS+=("$k" "$k-headers")
  # Any kernel with a suffix (e.g. linux-cachyos-bore) needs CachyOS repos
  [[ "$k" == linux-cachyos-* ]] && NEED_CACHYOS=1
done

# If any selected kernel needs CachyOS repos, add them to the live ISO now
if [ "$NEED_CACHYOS" -eq 1 ]; then
  echo "→ CachyOS kernel selected — adding CachyOS repos..."
  sudo pacman-key --recv-keys F3B607488DB35A47 --keyserver keyserver.ubuntu.com
  sudo pacman-key --lsign-key F3B607488DB35A47
  sudo pacman -U https://mirror.cachyos.org/repo/x86_64/cachyos/cachyos-keyring-20240331-1-any.pkg.tar.zst \
                  https://mirror.cachyos.org/repo/x86_64/cachyos/cachyos-mirrorlist-27-1-any.pkg.tar.zst \
                  https://mirror.cachyos.org/repo/x86_64/cachyos/cachyos-v3-mirrorlist-27-1-any.pkg.tar.zst \
                  https://mirror.cachyos.org/repo/x86_64/cachyos/cachyos-v4-mirrorlist-27-1-any.pkg.tar.zst
  sudo tee -a /etc/pacman.conf <<'EOF'

[cachyos]
Include = /etc/pacman.d/cachyos-mirrorlist
[cachyos-v3]
Include = /etc/pacman.d/cachyos-v3-mirrorlist
[cachyos-v4]
Include = /etc/pacman.d/cachyos-v4-mirrorlist
EOF
  sudo pacman -Syy
fi
```

<details>
<summary>📋 Kernel reference — what each one is best at</summary>

| Kernel | Repo | Scheduler | Best for |
|--------|------|-----------|----------|
| `linux-zen` | `[extra]` | EEVDF | Desktop/laptop daily use — lower latency, tuned for interactivity |
| `linux-cachyos` | `[extra]` | EEVDF | CachyOS defaults — balanced optimization with extra patches |
| `linux-cachyos-bore` | CachyOS | BORE | Gaming/audio — Burst-Oriented Response Enhancer prioritizes foreground tasks |
| `linux-cachyos-eevdf` | CachyOS | EEVDF | General desktop — EEVDF scheduler with CachyOS optimizations |
| `linux` | `[core]` | EEVDF | Maximum stability — vanilla kernel, least patches, slowest to adopt new features |
| `linux-lts` | `[core]` | EEVDF | Fallback kernel — older version, ultra-stable, ideal rescue boot option |

**Scheduler explainer:**
- **EEVDF** (Earliest Eligible Virtual Deadline First) — the default Linux scheduler since 6.6. Fair, predictable, good all-rounder.
- **BORE** (Burst-Oriented Response Enhancer) — prioritizes the currently-focused task. Noticeably snappier for single-app workloads (gaming, DAWs, video editing). Can slightly penalize heavy background tasks.

**CachyOS vs vanilla kernels:**
CachyOS kernels add patches for: x86-64-v3/v4 optimized code paths, BBRv3 TCP congestion control, AMD P-State EPP, LZ4 compression in the kernel, and various scheduler/MM tweaks. Available in `[extra]` — no third-party repo needed.

**Recommended approach:**
Install `linux-zen` as your daily driver and `linux-lts` as fallback. If you game or do real-time audio, add `linux-cachyos-bore` as a third option. GRUB picks the first kernel by default; hold Shift during boot to choose another.

</details>

### 3.2 vconsole

Sets the default TTY keymap and font. `ter-124n` is a high-DPI-friendly Terminus variant. This only affects the text-mode console (TTY), not your Wayland/X11 sessions:

```bash
cat <<'EOF' > /mnt/etc/vconsole.conf
KEYMAP=us
FONT=ter-124n
EOF
```

### 3.3 pacstrap

`pacstrap` bootstraps a new Arch system onto `/mnt`. It installs the package group `base`, your chosen kernel, firmware, essential tools, networking, SSH, and PipeWire for audio:

```bash
pacstrap -K /mnt \
  base base-devel \
  "${KERNEL_PKGS[@]}" linux-firmware "${cpu}-ucode" \
  efibootmgr btrfs-progs dosfstools e2fsprogs exfatprogs \
  networkmanager openssh \
  nvim git sudo man curl \
  zsh zsh-completions zsh-autosuggestions bash-completion tmux \
  pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber reflector

cp /etc/pacman.conf /mnt/etc/pacman.conf
cp /etc/pacman.d/mirrorlist /mnt/etc/pacman.d/mirrorlist
# Copy CachyOS mirrorlists if they were added in §3.1
cp /etc/pacman.d/cachyos*-mirrorlist /mnt/etc/pacman.d/ 2>/dev/null || true
cp /etc/pacman.d/cachyos-v*-mirrorlist /mnt/etc/pacman.d/ 2>/dev/null || true
```

> **What we're installing:**
> - `base base-devel` — core Arch system + build toolchain
> - `${KERNEL_PKGS[*]}` — your chosen kernel(s) + headers (needed for DKMS modules like NVIDIA)
> - `linux-firmware` + microcode — hardware firmware + CPU patches
> - `efibootmgr btrfs-progs dosfstools` — UEFI boot management + filesystem tools
> - `networkmanager openssh` — networking + remote access
> - Editors, git, shell tools — daily driver CLI
> - `pipewire wireplumber` — modern audio stack (replaces PulseAudio/JACK)
> - `reflector` — auto-updates pacman mirror list

### 3.4 Enter chroot

`arch-chroot` switches into the new system — `/mnt` becomes `/`. From here on, we're configuring the installed system, not the live ISO:

```bash
arch-chroot /mnt
```

### CachyOS Repos (optional)

> Skip if using Vanilla Arch repos. This should be the first thing you do after entering chroot — so every package from §4 onward pulls from CachyOS mirrors.

```bash
# If repos were already added in §3.1 (CachyOS kernel selected), skip this.
if ! grep -q cachyos /etc/pacman.conf 2>/dev/null; then
  sudo pacman -Syu
  cd ~
  curl https://mirror.cachyos.org/cachyos-repo.tar.xz -o cachyos-repo.tar.xz
  tar xf cachyos-repo.tar.xz && cd cachyos-repo
  sudo ./cachyos-repo.sh
  cd ~ && rm -rf cachyos-repo cachyos-repo.tar.xz

  # Reinstall everything from CachyOS repos
  # All packages installed so far (§3) were from vanilla Arch.
  # CachyOS packages have bumped pkgrel (1.2.3-1 → 1.2.3-1.1),
  # so pacman sees them as newer and upgrades automatically.
  sudo pacman -Qqn | sudo pacman -S --noconfirm -
fi

# Rank CachyOS mirrors by speed (both methods include this)
sudo pacman -S --noconfirm --needed cachyos-rate-mirrors
sudo cachyos-rate-mirrors
```

> After this, every package on the system is the CachyOS-optimized version.

---

## §4 — Chroot Configuration

We're now "inside" the new system. Everything from here through §7 runs in this chroot.

### 4.1 Safety Check

Verify we're actually in chroot (the root filesystem should be Btrfs, not the live ISO's overlay/squashfs):

```bash
[ "$(findmnt -n -o FSTYPE /)" = "btrfs" ] || { echo "ERROR: Not in chroot"; exit 1; }
```

### 4.2 Locale & Timezone

Locale determines language, date/number formatting, and character encoding. We generate four: US English (primary), British English, Japanese, and Thai:

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

The system's network name. Change `host_name` to whatever identifies this machine:

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

Create the root password, then your daily user. The `docker` group is pre-created so you can install Docker later without re-adding yourself to groups:

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

> The group memberships: `wheel` (sudo access), `storage` (disk management), `power` (shutdown/reboot), `audio` (sound), `video` (GPU/backlight), `docker` (container management).

### 4.5 mkinitcpio

`mkinitcpio` builds the initramfs — the minimal Linux system that loads at boot before your root filesystem is mounted. We configure it to include Btrfs support and handle resume from swap (hibernation):

```bash
perl -pi -e '
  s/^MODULES=\(\)/MODULES=(btrfs)/;
  s/^BINARIES=\(\)/BINARIES=(\/usr\/bin\/btrfs)/;
  s/^HOOKS=\(.*\)$/HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block encrypt filesystems resume fsck)/;
' /etc/mkinitcpio.conf

mkinitcpio -P
```

> **HOOKS explained:**
> - `base` — core initramfs infrastructure
> - `udev` — device detection
> - `autodetect` — shrinks initramfs by including only currently-used modules
> - `microcode` — applies CPU microcode before kernel init
> - `modconf` — loads modules from modprobe.d config
> - `kms` — early KMS (kernel mode setting) for flicker-free boot
> - `keyboard keymap consolefont` — early keyboard + font support (needed for LUKS passphrase prompts)
> - `block encrypt` — block device + encryption support
> - `filesystems` — mounts root (includes Btrfs detection)
> - `resume` — hibernation resume from swap
> - `fsck` — filesystem check

---

## §5 — Bootloader

> ⚠️ **Pick ONE bootloader below.** These are alternatives — do not install both.

### Clean Old EFI Entries (optional)

UEFI motherboards accumulate stale boot entries from old installs. List them, then delete any you don't need:

```bash
efibootmgr -v                         # list all entries
# efibootmgr -b XXXX -B               # delete entry by boot number (e.g. 0001, 0002)
```

> Keep the entry for your current bootloader. If you see old "Linux Boot Manager", "Windows Boot Manager" from a wiped disk, or duplicates — remove them. The boot order (`BootOrder`) auto-updates.

### ▸ GRUB

GRUB is the most widely-used Linux bootloader. It reads Btrfs directly, chain-loads from the ESP, and supports snapshot boot entries via `grub-btrfs`.

Install and deploy to the ESP:

```bash
pacman -S --noconfirm --needed grub
grub-install --target=x86_64-efi --efi-directory=/boot/EFI --bootloader-id=GRUB
```

Now configure the kernel command line — these parameters are passed to the kernel at every boot:

```bash
ucode_img="intel"
lscpu | grep -qi amd && ucode_img="amd"

swap_part="$(blkid -t TYPE=swap -o device | head -1)"
swap_uuid="$(blkid -s UUID -o value "$swap_part")"

sed -i '/^GRUB_CMDLINE_LINUX_DEFAULT=/d' /etc/default/grub
sed -i "/^GRUB_CMDLINE_LINUX=/a GRUB_CMDLINE_LINUX_DEFAULT=\"loglevel=3 resume=UUID=${swap_uuid} zswap.enabled=1 zswap.compressor=lz4 zswap.max_pool_percent=50 zswap.zpool=zsmalloc ${ucode_img}_iommu=on iommu=pt\"" /etc/default/grub
```

> **Kernel parameters explained:**
> - `loglevel=3` — only show errors and warnings (quieter boot)
> - `resume=UUID=...` — where to resume from for hibernation
> - `zswap.enabled=1` — enable compressed RAM cache for swap pages (faster than disk swap)
> - `zswap.compressor=lz4` — use LZ4 compression (fast, decent ratio)
> - `zswap.max_pool_percent=50` — max 50% of RAM used for compressed swap cache
> - `zswap.zpool=zsmalloc` — use zsmalloc allocator (efficient for compressed pages)
> - `iommu=pt` — IOMMU in passthrough mode (needed for GPU passthrough, safe default)
> - `${ucode_img}_iommu=on` — enable CPU IOMMU (Intel VT-d / AMD-Vi)

Add zswap compression modules to the initramfs so they're available immediately at boot:

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

Generate the GRUB configuration file and we're done:

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

> **Dual-boot (optional):** If you have Windows or another OS on the same disk, install `os-prober` and regenerate: `pacman -S --needed os-prober && echo 'GRUB_DISABLE_OS_PROBER=false' >> /etc/default/grub && grub-mkconfig -o /boot/grub/grub.cfg`

### ▸ Limine

Limine is a simpler, modern bootloader that works via the Limine boot protocol. It reads kernel + initramfs directly from the ESP (FAT32), so we copy artifacts there and generate a `limine.conf`.

Install and register with the UEFI firmware:

```bash
pacman -S --noconfirm --needed limine
mkdir -p /boot/EFI/limine /boot/limine
cp -v /usr/share/limine/*.EFI /boot/EFI/limine/

efibootmgr --create --disk /dev/nvme0n1 --part 1 \
  --label "Limine Bootloader" \
  --loader '\EFI\limine\BOOTX64.EFI' \
  --unicode
```

> `efibootmgr --create` adds a boot entry to your motherboard's NVRAM. The `--unicode` flag enables UTF-8 support in the boot menu.

Now copy kernel, initramfs, and microcode to the ESP, then generate a `limine.conf` entry for each kernel. The first kernel in your list becomes the default:

```bash
root_part="$(blkid -t TYPE=btrfs -o device | head -1)"
swap_part="$(blkid -t TYPE=swap -o device | head -1)"
root_uuid="$(blkid -s UUID -o value "$root_part")"
swap_uuid="$(blkid -s UUID -o value "$swap_part")"

ucode_img="intel"
lscpu | grep -qi amd && ucode_img="amd"

# Copy artifacts
cp -v "/boot/${ucode_img}-ucode.img" /boot/limine/

# Generate limine.conf — one entry per kernel
cat > /boot/limine/limine.conf <<LIMINE_HEADER
TIMEOUT=3
DEFAULT_ENTRY=Arch Linux (${KERNELS[0]})

LIMINE_HEADER

for k in "${KERNELS[@]}"; do
  # Main entry
  cat >> /boot/limine/limine.conf <<BLOCK
/Arch Linux (${k})
    PROTOCOL: linux
    KERNEL_PATH: boot():/limine/vmlinuz-${k}
    MODULE_PATH: boot():/limine/${ucode_img}-ucode.img
    MODULE_PATH: boot():/limine/initramfs-${k}.img
    CMDLINE: loglevel=3 root=UUID=${root_uuid} rootflags=subvol=@ rootfstype=btrfs rw resume=UUID=${swap_uuid} zswap.enabled=1 zswap.compressor=lz4 zswap.max_pool_percent=50 zswap.zpool=zsmalloc ${ucode_img}_iommu=on iommu=pt

/Arch Linux (${k} fallback)
    PROTOCOL: linux
    KERNEL_PATH: boot():/limine/vmlinuz-${k}
    MODULE_PATH: boot():/limine/initramfs-${k}-fallback.img
    CMDLINE: loglevel=3 root=UUID=${root_uuid} rootflags=subvol=@ rootfstype=btrfs rw
BLOCK
done

cat /boot/limine/limine.conf  # sanity check
```

> The `boot():` prefix means "the partition containing this config file" (the ESP). Each kernel gets two entries — main (with microcode, zswap, IOMMU) and fallback (stripped down, using the larger `-fallback.img` for rescue).

Pacman hook to keep Limine EFI files in sync after updates:

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

Add zswap modules and rebuild (same as GRUB), then re-copy:

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

> ⚠️ **Remember:** After any kernel update, you must re-copy the new initramfs to `/boot/limine/`. The Limine hook only copies its own EFI binary, not kernel artifacts. Consider `limine-mkinitcpio-hook` (AUR) for automation.

---

## §6 — Services & QoL

These are the background services that keep your system running. NetworkManager handles all networking (Wi-Fi, Ethernet, VPN), bluetoothd handles Bluetooth, reflector updates your mirror list weekly, sshd lets you SSH in, and fstrim keeps your SSD healthy:

### 6.1 Extra Packages

```bash
pacman -Syu --noconfirm --needed \
  util-linux inetutils usbutils rsync htop bat zip unzip p7zip \
  avahi nss-mdns \
  alsa-utils sof-firmware easyeffects \
  bluez bluez-utils \
  xdg-user-dirs
```

> - `sof-firmware` — audio DSP firmware for modern Intel/AMD laptops
> - `easyeffects` — PipeWire audio effects (EQ, compression, etc.)
> - `avahi nss-mdns` — mDNS (`.local` hostname resolution, printer discovery)
> - `xdg-user-dirs` — creates standard folders (Desktop, Documents, Downloads, etc.)

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
# pacman -S --needed acpi  # battery status CLI (systemd-logind handles ACPI events now)
```

> - `reflector.timer` — weekly mirror list refresh (keeps downloads fast)
> - `fstrim.timer` — weekly SSD TRIM (maintains performance)
> - `sshd` — SSH server (we enabled password auth during install; harden in §9.6)

---

## §7 — Desktop Stack

> ⚠️ **Pick ONE path.** KDE is all-in-one — install it and stop. Niri needs a shell — install Niri then Noctalia. Do not install both KDE and Niri.

There are two philosophies:

| Path | What to install | What you get | Login screen |
|------|----------------|-------------|--------------|
| 🖥️ **KDE Plasma** | Just the KDE section | Full desktop — compositor, shell, apps, all integrated | `plasma-login-manager` |
| 🏔️ **Niri + Noctalia** | Niri section + Noctalia section | Scrollable-tiling compositor + native shell | `greetd` + `noctalia-greeter` |
| 💀 **Niri alone** | Just the Niri section | Bare compositor — no bar, no launcher, no wallpaper. You build the rest yourself. | none (start from TTY) |

KDE Plasma is the mainstream choice: everything works out of the box, familiar desktop metaphor, KDE apps integrate perfectly. Niri + Noctalia is leaner: tiling workflow, lower resource usage, keyboard-driven, but still has a full shell with bar/launcher/notifications. Niri alone is for people who want to hand-pick every component (waybar, fuzzel, swaybg, etc.) — the guide doesn't cover that.

### ▸ KDE Plasma

Plasma 6.6+ ships `plasma-login-manager` as its native login screen (replaces SDDM). We also need `xdg-desktop-portal` for Flatpak/Snap integration and screen sharing:

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

> - `plasma-nm` — NetworkManager applet (Wi-Fi selection, VPN)
> - `plasma-pa` — audio volume applet
> - `kscreen` — monitor hotplug handling
> - `kde-gtk-config breeze-gtk` — makes GTK apps (Firefox, GIMP) match KDE's theme
> - `power-profiles-daemon` — laptop power modes (balanced/powersave/performance)

**PAM config — MANDATORY.** `plasma-login-manager` does not ship a default PAM file. Without this, the login screen cannot authenticate anyone:

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

> `include system-login` pulls in `pam_unix.so` — the actual password checker. Without this file, PLM has no authentication chain and login always fails with "Authentication for user '' failed".

**Desktop Apps:**

```bash
pacman -S --noconfirm --needed \
  dolphin dolphin-plugins konsole kate okular gwenview spectacle ark \
  gparted kio-extras ffmpegthumbs kdegraphics-thumbnailers \
  filelight kcalc btop fastfetch
```

> - `kio-extras` — SMB/SFTP/FTP support inside Dolphin's address bar
> - `ffmpegthumbs` — video thumbnails in Dolphin
> - `kdegraphics-thumbnailers` — PDF/PS/RAW photo thumbnails

**KWallet (optional):** Stores passwords for KDE apps, VS Code, Git credential helpers, and browser password sync:

```bash
pacman -S --noconfirm --needed kwallet kwalletmanager kwallet-pam
```

> Auto-unlock requires: wallet password = login password, blowfish encryption, wallet name = `kdewallet`.

### ▸ Compositor: Niri

Niri is a scrollable-tiling Wayland compositor — windows arrange in columns that scroll horizontally. It's in the official `[extra]` repo. The shell (Noctalia) is AUR and gets installed post-reboot.

> 💡 **Terminal & file manager:** This guide uses `ghostty` (GPU-accelerated terminal) and `dolphin` (KDE file manager) — these are my personal preferences. Niri's factory default terminal is `alacritty`, and the typical GNOME-style file manager would be `nautilus`. Swap as you like.

```bash
pacman -S --noconfirm --needed niri
```

**Session file** — tells the display manager (greetd) how to launch this session:

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

> `niri-session` (not plain `niri`) exports environment variables to systemd correctly — this matters for `xdg-desktop-portal` and other D-Bus services.

**Base Wayland + integration packages:**

```bash
pacman -S --noconfirm --needed \
  xdg-desktop-portal xdg-desktop-portal-gnome xdg-desktop-portal-gtk \
  qt6-wayland xorg-xwayland xwayland-satellite \
  xdg-utils shared-mime-info kde-cli-tools \
  accountsservice qt5-wayland \
  gvfs-mtp gvfs-gphoto2 udiskie \
  plasma-integration kded qt6ct-kde \
  noto-fonts-emoji wl-clipboard adw-gtk-theme \
  ghostty libcanberra
```

> Key packages:
> - `xdg-desktop-portal-gnome` + `-gtk` — screen sharing for OBS, Discord, browsers (GNOME portal works better on non-KDE compositors)
> - `wl-clipboard` — `wl-copy` / `wl-paste` CLI clipboard for Wayland (scripts, terminal workflows)
> - `adw-gtk-theme` — GTK apps (Firefox, VS Code) look consistent with the system theme
> - `noto-fonts-emoji` — emoji rendering in terminal, browser, and GTK apps
> - `xwayland-satellite` — run X11 apps under Wayland (multi-window support)
> - `plasma-integration kded` — KDE file dialogs, trash support, MIME type handling
> - `qt6ct-kde` — apply KDE color schemes to Qt apps when Plasma isn't running
> - `ghostty` — GPU-accelerated terminal emulator (we use this instead of Konsole)
> - `libcanberra` — freedesktop sound feedback (volume change dings)

**Desktop Apps — same as KDE, minus Konsole (ghostty replaces it):**

```bash
pacman -S --noconfirm --needed \
  dolphin dolphin-plugins kate okular gwenview spectacle ark \
  gparted kio-extras ffmpegthumbs kdegraphics-thumbnailers \
  filelight kcalc btop fastfetch capitaine-cursors
```

> Without `plasma-integration` + `kded6` running (autostarted in the Noctalia section), KDE apps would use ugly fallback themes, lack file dialogs, and have broken trash support.
>
> **If stopping here (Niri alone):** You now have a bare compositor. Start Niri from TTY with `niri-session`. Install a bar (`waybar`), launcher (`fuzzel`), wallpaper setter (`swaybg`), and notification daemon (`mako`) separately. This guide doesn't cover that path. For a complete desktop, continue to the Noctalia section.

### ▸ Shell: Noctalia v5

Noctalia v5 is a native C++ desktop shell. It provides the bar, app launcher, dock, notifications, wallpaper, OSD, clipboard manager, night light, and lock screen — all the things a desktop environment typically bundles, but as a compositor-agnostic layer.

Install post-reboot (AUR — needs `yay` from §9.1):

```bash
# After reboot + yay installed (§9.1):
yay -S --noconfirm --needed noctalia-git noctalia-greeter
sudo pacman -S --noconfirm --needed greetd
```

**greetd** is a minimal login daemon. We configure it to use `noctalia-greeter` as its frontend:

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

> `greetd` runs on virtual terminal 1 (VT 1). The greeter runs as the `greetd` user for security — it never sees your password directly, only passes it to PAM.

**Niri config** (`~/.config/niri/config.kdl`):

This is the complete, bootable configuration. The config language is KDL (a document language, not a programming language — no variables, no logic):

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

> **Full config reference:** See companion guide [`Niri_Noctalia_v5.md`](Niri_Noctalia_v5.md) for animations, gaming window rules, modular config splitting, greeter reference, and troubleshooting.

#### Secret Storage

Apps need a secrets backend to safely store passwords. GTK apps (VS Code, Chromium, Firefox, Git) use `libsecret`. KDE apps (Dolphin network passwords, KDE Connect) use KWallet. Install both — they coexist fine:

```bash
sudo pacman -S --noconfirm --needed gnome-keyring libsecret kwallet kwalletmanager kwallet-pam

# PAM hooks for greetd (auto-unlock at login)
sudo tee -a /etc/pam.d/greetd <<'EOF'
auth       optional     pam_gnome_keyring.so
session    optional     pam_gnome_keyring.so auto_start
session    optional     pam_kwallet5.so auto_start kwalletd=/usr/bin/ksecretd
EOF
```

> KWallet auto-unlock: wallet password = login password, blowfish encryption, wallet name = `kdewallet`.

---

## §8 — Reboot

Time to leave the installer and boot into the real system:

```bash
exit          # exit chroot
umount -R /mnt
swapoff -a
reboot
```

Remove the USB drive when prompted. Log in as your user.

---

## §9 — Post-Install

Everything below runs on the new system, logged in as your user.

### 9.1 XDG User Dirs

Creates `~/Desktop`, `~/Documents`, `~/Downloads`, etc.:

```bash
xdg-user-dirs-update
```

### 9.2 YAY (AUR Helper)

`yay` is a pacman wrapper that also handles the Arch User Repository. It builds packages from source using PKGBUILD scripts:

```bash
sudo pacman -S --noconfirm --needed git base-devel
git clone https://aur.archlinux.org/yay.git && cd yay && makepkg -si
cd .. && rm -rf yay
```

> **AUR Security:** The AUR is community-maintained. Always review PKGBUILDs — look for suspicious source URLs, obfuscated commands, or curl-pipe-shell patterns. Packages marked 🔒 below need manual inspection before installing.

### 9.3 CachyOS Extras (optional)

> Skip if using Vanilla Arch repos. The CachyOS repos were added [after §5](#cachyos-repos-optional).

```bash
# Core CachyOS optimizations (always recommended if using CachyOS repos)
sudo pacman -Syu
sudo pacman -S cachyos-settings
sudo systemctl enable --now ananicy-cpp

# Optional: gaming meta-package (gamemode, steam, lutris, mangohud, proton-ge...)
# sudo pacman -S cachyos-gaming-meta

# Optional: welcome app (system info, quick links, post-install tips)
# sudo pacman -S cachyos-hello

# Optional: userspace OOM killer — kills bloated apps before system freezes,
# more responsive than kernel OOM. Enable after installing systemd-oomd:
# sudo pacman -S systemd-oomd
# sudo systemctl enable --now systemd-oomd
```

> `cachyos-settings` pulls in `ananicy-cpp` (auto process priority — games/media get higher priority, background tasks lower), `zram-generator` (compressed RAM swap), and CachyOS-specific defaults. `cachyos-gaming-meta` is a convenience bundle — you can also install gaming packages individually in §9.9.

### 9.4 GPU Driver

Auto-detect your GPU and install the right driver:

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

`nvidia-open-dkms` is the open-source kernel module (610.x+). It works with all kernels including linux-zen and linux-cachyos because it builds against your running kernel via DKMS:

```bash
if [ "$gpu_vendor" = "nvidia" ]; then
  sudo pacman -S --noconfirm --needed \
    nvidia-open-dkms nvidia-utils lib32-nvidia-utils \
    nvidia-settings libxnvctrl \
    ocl-icd opencl-nvidia lib32-opencl-nvidia clinfo cuda

  # Gaming extras — VA-API, Vulkan, DXVK
  sudo pacman -S --noconfirm --needed \
    libva-utils vdpauinfo vulkan-tools \
    libva-nvidia-driver dxvk vkd3d shaderc spirv-tools
```

NVIDIA needs Wayland DRM modesetting enabled on the kernel command line. For GRUB:

```bash
  if grep -q 'GRUB_CMDLINE_LINUX_DEFAULT' /etc/default/grub 2>/dev/null; then
    sudo sed -i '/^GRUB_CMDLINE_LINUX_DEFAULT=/s/"$/ nvidia-drm.modeset=1 nvidia-drm.fbdev=1"/' /etc/default/grub
    sudo grub-mkconfig -o /boot/grub/grub.cfg
  fi
```

> For Limine: add `nvidia-drm.modeset=1 nvidia-drm.fbdev=1` to the `CMDLINE:` line in `/boot/limine/limine.conf`.

Configure mkinitcpio: add NVIDIA modules, remove the generic `kms` hook (NVIDIA provides its own):

```bash
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
```

Pacman hook — auto-rebuilds initramfs whenever the NVIDIA driver or kernel is updated:

```bash
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

**Intel iGPU / Arc dGPU (if detected):**

Intel graphics drivers are built into the kernel — no kernel module to install. But you'll want Vulkan, hardware video decode, and verification tools:

```bash
if [ "$gpu_vendor" = "intel" ]; then
  sudo pacman -S --noconfirm --needed \
    vulkan-intel intel-media-driver \
    mesa-utils libva-utils vulkan-tools
fi
```

> - `vulkan-intel` — Intel ANV Vulkan driver (games, GPU compute)
> - `intel-media-driver` — hardware video encode/decode (VA-API — YouTube, OBS, video players)
> - `mesa-utils` — `glxinfo` for OpenGL verification
> - `libva-utils` — `vainfo` to verify hardware video support
> - `vulkan-tools` — `vkcube`, `vulkaninfo` for Vulkan verification

### 9.5 Snapper

Snapper manages Btrfs snapshots — point-in-time copies of your subvolumes. Combined with `snap-pac` (automatic pre/post snapshots on every `pacman` transaction) and `grub-btrfs` (boot into snapshots from GRUB), you get a safety net for system updates:

```bash
# Install
sudo pacman -S --noconfirm --needed snapper btrfs-assistant
yay -S --noconfirm --needed grub-btrfs snap-pac snap-pac-grub snapper-gui-git  # 🔒
```

Create configs — one per subvolume you want to snapshot:

```bash
sudo snapper -c root create-config /
sudo snapper -c home create-config /home   # skip if you don't want /home snapshots

# GRUB only — /boot is on Btrfs so we can snapshot it too
sudo snapper -c boot create-config /boot
sudo systemctl enable --now grub-btrfsd
```

**Retention settings.** `snap-pac` hooks already create pre/post snapshots on every `pacman` transaction — that covers `/` and `/boot` for all package updates (kernel, drivers, system tools). Timeline snapshots on top of that are redundant. Just set number limits as a hard cap:

```bash
# root — no timeline, snap-pac handles package updates
sudo snapper -c root set-config TIMELINE_CREATE=no
sudo snapper -c root set-config NUMBER_LIMIT=15 NUMBER_LIMIT_IMPORTANT=5

# home — optional light timeline (user files aren't covered by snap-pac)
sudo snapper -c home set-config TIMELINE_CREATE=yes
sudo snapper -c home set-config TIMELINE_LIMIT_HOURLY=0 TIMELINE_LIMIT_DAILY=3 TIMELINE_LIMIT_WEEKLY=2
sudo snapper -c home set-config NUMBER_LIMIT=10 NUMBER_LIMIT_IMPORTANT=3

# boot (GRUB) — no timeline needed, snap-pac captures kernel updates
sudo snapper -c boot set-config TIMELINE_CREATE=no 2>/dev/null
sudo snapper -c boot set-config NUMBER_LIMIT=5 NUMBER_LIMIT_IMPORTANT=3 2>/dev/null
```

> `IMPORTANT` snapshots are pre/post pairs from `snap-pac`. Number limits keep them from eating your disk.

Enable the timers that create and clean up snapshots:

```bash
sudo systemctl enable --now snapper-timeline.timer snapper-cleanup.timer
```

### 9.6 SSH Hardening

> ⚠️ **Do this after setting up SSH keys** (§9.2 client keygen + `ssh-copy-id`). Otherwise you'll lock yourself out.

```bash
sudo tee -a /etc/ssh/sshd_config <<'EOF'

# ── Authentication ──
PermitRootLogin no              # never allow root SSH
PasswordAuthentication no       # keys only
KbdInteractiveAuthentication no # disable keyboard-interactive
PermitEmptyPasswords no         # just in case

# ── Brute-force protection ──
MaxAuthTries 3                  # default: 6
LoginGraceTime 60               # seconds to authenticate (default: 120)

# ── Idle timeout ──
ClientAliveInterval 300         # send keepalive every 5 min
ClientAliveCountMax 2           # disconnect after 2 missed (10 min)

# ── Optional: change port (security through obscurity) ──
# Port 2222
EOF

# If you changed the port, uncomment and restart:
# sudo sed -i 's/^#Port 22/Port 2222/' /etc/ssh/sshd_config

sudo systemctl restart sshd
```

> After changing the port: `ssh -p 2222 user@host`. Update `~/.ssh/config` on your client with `Port 2222` under the host entry.

### 9.7 Firewall

`ufw` is a simple frontend for `iptables`/`nftables`. Default deny incoming, allow SSH:

```bash
sudo pacman -S --noconfirm --needed ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
# If you changed the SSH port:
# sudo ufw allow 2222/tcp
sudo ufw enable
sudo systemctl enable ufw
```

> For KDE Connect, gaming (Steam), or local dev servers, add specific rules as needed. Desktop firewalls are mostly defense-in-depth — your router already blocks inbound traffic.

### 9.8 AppArmor (optional)

AppArmor restricts what each application can do — it's Mandatory Access Control (MAC) like SELinux but simpler. Enable it with a kernel parameter, then install profiles:

```bash
# 1. Install
sudo pacman -S --noconfirm --needed apparmor

# 2. Kernel parameter (GRUB)
sudo sed -i '/^GRUB_CMDLINE_LINUX_DEFAULT=/s/"$/ lsm=landlock,lockdown,yama,apparmor,bpf"/' /etc/default/grub
sudo grub-mkconfig -o /boot/grub/grub.cfg

# 3. Enable and reboot
sudo systemctl enable apparmor
```

> After reboot, check: `sudo aa-status`. AppArmor needs profile packages for each app — start with `apparmor-profiles` from AUR. This is advanced; skip if you just want a working desktop.

### 9.9 Extra Packages & Fonts

Personal pick list — install what you need:

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
# noto-fonts-emoji already installed if you chose Niri (§7.2)
sudo pacman -S --noconfirm --needed \
  noto-fonts noto-fonts-emoji \
  ttf-dejavu ttf-ubuntu-font-family \
  terminus-font nerd-fonts ttf-ms-fonts

# Flatpak (optional — for sandboxed apps)
sudo pacman -S --noconfirm --needed flatpak
```

### 9.10 pyenv

`pyenv` manages multiple Python versions per-user without conflicting with the system Python:

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

### 9.11 SPDIF Audio Fix (optional)

Some SPDIF DACs sleep after idle → first 1–3 seconds of audio get cut off. Two fixes:

```bash
# 1. Disable codec autosuspend (system-wide)
sudo tee /etc/modprobe.d/alsa-no-powersave.conf >/dev/null <<'EOF'
options snd_hda_intel power_save=0 power_save_controller=N
EOF

# 2. Stop WirePlumber from suspending sinks (per-user)
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

### 9.12 Cache Cleanup

Clear pacman's package cache to reclaim disk space:

```bash
sudo pacman -Scc
```

---

## Credits

Huge thanks to the authors/maintainers of these resources:

- `fstab` notes: <https://gist.github.com/mjkstra/96ce7a5689d753e7a6bdd92cdc169bae#fstab>
- Arch install walkthrough/reference: <https://github.com/silentz/arch-linux-install-guide>
- Chroot exit + reboot checklist: <https://gist.github.com/yovko/512326b904d120f3280c163abfbcb787#exit-from-chroot-and-reboot>
- NVIDIA driver guide: <https://github.com/korvahannu/arch-nvidia-drivers-installation-guide>
- Noctalia v5 docs: <https://docs.noctalia.dev/v5/>
- Niri Arch Wiki: <https://wiki.archlinux.org/title/Niri>
- CachyOS Niri config patterns: <https://github.com/CachyOS/cachyos-niri-noctalia>
- [Arch Wiki](https://wiki.archlinux.org/)
- [Limine](https://github.com/limine-bootloader/limine)
- [CachyOS](https://cachyos.org/)

*Last updated: July 2026*
