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
  - [▸ 2.2 Common Mounts](#-22-common-mounts)
  - [▸ 2.3 Boot & ESP Mount (per-bootloader)](#-23-boot--esp-mount-per-bootloader)
  - [2.4 fstab](#24-fstab)
- [§3 — Base Install](#3--base-install)
  - [3.0 CPU Detection](#30-cpu-detection)
  - [3.0a Optional: Enable CachyOS Repos](#30a-optional-enable-cachyos-repos-live-iso)
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
  - [6.3 YAY (AUR Helper)](#63-yay-aur-helper)
- [§7 — Desktop Stack](#7--desktop-stack)
  - [▸ Secret Storage (all paths)](#-secret-storage-all-paths)
- [§8 — Reboot](#8--reboot)
- [§9 — Post-Install](#9--post-install)
  - [9.1 XDG User Dirs](#91-xdg-user-dirs)
  - [9.2 CachyOS Extras](#92-cachyos-extras-optional)
  - [9.3 GPU Driver](#93-gpu-driver)
  - [9.4 Snapper](#94-snapper)
  - [9.5 SSH Hardening](#95-ssh-hardening)
  - [9.6 Firewall](#96-firewall)
  - [9.7 AppArmor](#97-apparmor-optional)
  - [9.8 Extra Packages & Fonts](#98-extra-packages--fonts)
  - [9.9 pyenv](#99-pyenv)
  - [9.10 SPDIF Audio Fix](#910-spdif-audio-fix-optional)
  - [9.11 Cache Cleanup](#911-cache-cleanup)
- [Credits](#credits)

---

## Decision Matrix

Choose ONE per row (multiple kernels OK). Each choice maps to the section where it takes effect.

| # | Decision | A | B | C | § |
|---|----------|---|---|---|---|
| 1 | **Kernel** | `linux-zen` | `linux-cachyos` *(only via §3.0a, up front — no post-chroot swap)* | `linux` / `linux-lts` | §3 |
| 2 | **Repos** | Vanilla Arch | CachyOS repos *(packages only — kernels must be chosen up front)* | | §3.0a (live ISO) or CachyOS Repos (post-chroot) |
| 3 | **Desktop** | KDE Plasma | Niri + Noctalia | Hyprland + Noctalia *(scrolling or tiling — see §7)* | §7 |
| 4 | **Bootloader** | GRUB | Limine | | §1,§2,§5 |

> **Kernel & repos — mostly independent.** `linux-zen`, `linux`, `linux-lts` work on Vanilla Arch. **Any `linux-cachyos*` kernel requires the CachyOS repos** — none are in the official `[extra]` repo, and a kernel can only be picked up front: enable the repos on the live ISO ([§3.0a](#30a-optional-enable-cachyos-repos-live-iso)) and pacstrap a `linux-cachyos*` kernel directly in §3.1. **CachyOS repos post-chroot** ([CachyOS Repos](#cachyos-repos-optional)) still give you CachyOS-optimized binaries for other packages — there is just no kernel swap.

### Recommended Combos

These are the three configurations this guide has been battle-tested with:

| Combo | Kernel | Repos | Desktop | Bootloader |
|-------|--------|-------|---------|------------|
| ⭐ **KDE Daily** | linux-zen | Vanilla | KDE Plasma | GRUB |
| 🏔️ **Niri Lean** | linux-zen | Vanilla | Niri+Noctalia | GRUB |
| 🪟 **Hyprland Scrolling** | linux-zen | Vanilla | Hyprland + Noctalia (Scrolling) | GRUB |
| 🪟 **Hyprland Tiling** | linux-zen | Vanilla | Hyprland + Noctalia (Tiling) | GRUB |
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

Arch ships with a global mirror list. Filter to nearby mirrors (Thailand + Singapore) ranked by speed — `-f 10` keeps the 10 fastest, `-l 10` only considers mirrors synced in the last 10 hours. `reflector` ships in the live ISO:

```bash
reflector -c Thailand,Singapore -f 10 -l 10 > /etc/pacman.d/mirrorlist
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

### ▸ 2.2 Common Mounts

The GRUB and Limine paths share almost everything — only the boot/ESP handling differs. So we do the shared part once here, then the bootloader-specific part in §2.3.

We create both `@boot` (GRUB: snapshotted kernels via `snap-pac`) and `@srv` (Limine: server data separation) up front — creating the unused one is harmless, only the mount below matters:

```bash
btrfs subvolume create /mnt/@boot
btrfs subvolume create /mnt/@srv

umount -R /mnt
mount -o compress=zstd:1,noatime,subvol=@ UUID="${root_uuid}" /mnt
mount --mkdir -o compress=zstd:1,noatime,subvol=@home     UUID="${root_uuid}" /mnt/home
mount --mkdir -o compress=zstd:1,noatime,subvol=@var_log  UUID="${root_uuid}" /mnt/var/log
mount --mkdir -o compress=zstd:1,noatime,subvol=@var_cache UUID="${root_uuid}" /mnt/var/cache
mount --mkdir -o compress=zstd:1,noatime,subvol=@var_tmp   UUID="${root_uuid}" /mnt/var/tmp
mount --mkdir -o compress=zstd:1,noatime,subvol=@root     UUID="${root_uuid}" /mnt/root
swapon UUID="${swap_uuid}"
```

> `@boot` is only used by GRUB, `@srv` only by Limine. Creating both costs nothing on a fresh install; skip the line for the one your bootloader won't use if you prefer a leaner subvolume list.

### ▸ 2.3 Boot & ESP Mount (per-bootloader)

Everything above is identical — here is where GRUB and Limine diverge. Pick the block for your bootloader from the [Decision Matrix](#decision-matrix).

**GRUB** — reads Btrfs natively, so `/boot` stays on a `@boot` Btrfs subvolume (kernel updates get snapshotted). The ESP is mounted *inside* it at `/boot/EFI`:

```bash
mount --mkdir -o compress=zstd:1,noatime,subvol=@boot UUID="${root_uuid}" /mnt/boot
mount --mkdir UUID="${esp_uuid}" /mnt/boot/EFI
```

> The ESP is mounted at `/boot/EFI` — inside the Btrfs `/boot`. GRUB reads the kernel from Btrfs `/boot`, then chain-loads from the FAT32 ESP at `/boot/EFI`.

**Limine** — can't read Btrfs, so the ESP is mounted directly at `/boot`. No `@boot` mount (kernel artifacts get copied to FAT32 in the [Bootloader section](#5--bootloader)):

```bash
mount --mkdir UUID="${esp_uuid}" /mnt/boot
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
- `linux` — vanilla stable kernel. Conservative, well-tested.
- `linux-lts` — long-term support. Older but extremely stable. Good fallback.
- `linux-cachyos*` (cachyos / bore / eevdf / bmq / lts / deckify / hardened) — needs the CachyOS repos, which can only be resolved from the live ISO ([§3.0a](#30a-optional-enable-cachyos-repos-live-iso)). If you want a CachyOS kernel, decide now — there is no post-chroot kernel swap.

> **Decision: Repos.** Vanilla Arch (`[core]`, `[extra]`, `[multilib]`) vs adding CachyOS repos. Two entry points:
> - **CachyOS-first** — enable the repos on the live ISO ([§3.0a](#30a-optional-enable-cachyos-repos-live-iso)) before §3.1, then pacstrap installs a `linux-cachyos*` kernel directly. The only way to get a CachyOS kernel.
> - **Post-chroot (default)** — start vanilla (any kernel from §3.1), then install the repos after chroot in the [CachyOS Repos](#cachyos-repos-optional) section for CachyOS-optimized binaries of other packages. Note: this does **not** change your kernel — pick a `linux-cachyos*` kernel up front if you want one.

### 3.0a Optional: Enable CachyOS Repos (Live ISO)

Do this **only** if you want a `linux-cachyos*` kernel installed from the start (pure CachyOS route) — it's the only way to get a CachyOS kernel. This adds the repos to the **live ISO's** pacman so §3.1/§3.3 can resolve them. Skipping it means you get a vanilla kernel and CachyOS repos later ([CachyOS Repos](#cachyos-repos-optional)) only for non-kernel packages.

> **Don't run `cachyos-repo.sh` here.** That script (a) installs CachyOS's forked pacman and (b) does a full package reinstall — both are **chroot operations** for the [CachyOS Repos](#cachyos-repos-optional) section. On the live ISO we only add the repos themselves (keyring + mirrorlists + pacman.conf), which is all pacstrap needs.

```bash
# Only if you want a linux-cachyos* kernel from pacstrap.
# 1. Trust the CachyOS signing key
sudo pacman-key --recv-keys F3B607488DB35A47 --keyserver keyserver.ubuntu.com
sudo pacman-key --lsign-key F3B607488DB35A47

# 2. Install keyring + mirrorlists (ALL THREE needed — v3/v4 tiers + base)
#    Resolves the latest version from the live repo — no hardcoded pins to go stale
PKG_BASE="https://mirror.cachyos.org/repo/x86_64/cachyos"
for p in cachyos-keyring cachyos-mirrorlist cachyos-v3-mirrorlist cachyos-v4-mirrorlist; do
  url="$PKG_BASE/$(curl -s "$PKG_BASE/" | grep -oE "${p}-[0-9][^\"<]*\.pkg\.tar\.zst" | sort -V | tail -1)"
  sudo pacman -U --noconfirm "$url"
done

# 3. Enable the repos in the live ISO's pacman.conf
sudo tee -a /etc/pacman.conf <<'EOF'

[cachyos]
Include = /etc/pacman.d/cachyos-mirrorlist
[cachyos-v3]
Include = /etc/pacman.d/cachyos-v3-mirrorlist
[cachyos-v4]
Include = /etc/pacman.d/cachyos-v4-mirrorlist
EOF

# 4. Refresh the live ISO's package DB with the new repos
sudo pacman -Syy
```

> **After §3.3 pacstrap, copy the CachyOS mirrorlists into the chroot yourself.** pacstrap only copies `/etc/pacman.d/mirrorlist` (the Arch default) and `/etc/pacman.conf` into `/mnt` — it does **not** copy the separate `cachyos-*-mirrorlist` files. Without them, the copied `pacman.conf` references `Include = /etc/pacman.d/cachyos-*-mirrorlist` that don't exist inside the chroot, and pacman there can't reach the repos. See §3.3 for the exact `cp`.

### 3.0 CPU Detection

The `*-ucode` package loads CPU microcode updates at boot — critical for security and stability:

```bash
cpu=intel
lscpu | grep -qi amd && cpu=amd
```

### 3.1 Select Kernels

Install at least one. You can install multiple — common combos: `linux-zen` (daily) + `linux-lts` (fallback).

> `linux-cachyos*` kernels need the CachyOS repos. If you enabled them on the live ISO ([§3.0a](#30a-optional-enable-cachyos-repos-live-iso)), you can pick a CachyOS kernel here and pacstrap installs it directly. Otherwise stay with one of the vanilla kernels above — the CachyOS repos (and their kernels) are only available from the live ISO, so a `linux-cachyos*` kernel must be chosen here if you want one.

```bash
# Uncomment the kernels you want. You can install several - GRUB picks the first.
# Vanilla (all in Arch repos, always available):
#   linux-zen (interactivity), linux (stable), linux-lts (long-term),
#   linux-hardened (security), linux-rt (real-time)
# CachyOS variants (linux-cachyos*) only resolve if you enabled §3.0a on the live ISO:
#   linux-cachyos, linux-cachyos-bore, linux-cachyos-eevdf, linux-cachyos-bmq,
#   linux-cachyos-lts, linux-cachyos-deckify, linux-cachyos-hardened
KERNELS=(
  linux-zen
  # linux
  # linux-lts
  # linux-hardened
  # linux-rt
  # linux-cachyos
  # linux-cachyos-bore
  # linux-cachyos-hardened
)

# Build kernel package list for pacstrap
KERNEL_PKGS=()
for k in "${KERNELS[@]}"; do
  KERNEL_PKGS+=("$k" "$k-headers")
done
```

<details>
<summary>📋 Kernel reference — what each one is best at</summary>

| Kernel | Repo | Scheduler | Best for |
|--------|------|-----------|----------|
| `linux-zen` | `[extra]` | EEVDF | Desktop/laptop daily use — lower latency, tuned for interactivity |
| `linux` | `[core]` | EEVDF | Maximum stability — vanilla kernel, least patches, slowest to adopt new features |
| `linux-lts` | `[core]` | EEVDF | Fallback kernel — older version, ultra-stable, ideal rescue boot option |
| `linux-hardened` | `[extra]` | EEVDF | Security — hardened malloc/stack & other hardening patches (slightly slower) |
| `linux-rt` | `[extra]` | EEVDF (PREEMPT_RT) | Real-time (PREEMPT_RT) — pro audio / MIDI / latency-critical, clock source accuracy |
| `linux-cachyos` | CachyOS repos | EEVDF | CachyOS defaults — balanced optimization with extra patches (best all-round) |
| `linux-cachyos-bore` | CachyOS repos | BORE | Gaming/audio — Burst-Oriented Response Enhancer prioritizes foreground tasks |
| `linux-cachyos-eevdf` | CachyOS repos | EEVDF | General desktop — explicit EEVDF build (in case default scheduler changes) |
| `linux-cachyos-bmq` | CachyOS repos | BMQ | Niche — BitMap Queue scheduler, heavy multi-thread workloads (no sched-ext) |
| `linux-cachyos-lts` | CachyOS repos | EEVDF | CachyOS long-term support — stable fallback / second kernel |
| `linux-cachyos-deckify` | CachyOS repos | BORE | Steam Deck & handheld tuning — gaming hardware patches (BORE) |
| `linux-cachyos-hardened` | CachyOS repos | BORE | Hardened + BORE — very aggressive security (large performance hit) |

**Scheduler explainer:**
- **EEVDF** (Earliest Eligible Virtual Deadline First) — the default Linux scheduler since 6.6. Fair, predictable, good all-rounder.
- **BORE** (Burst-Oriented Response Enhancer) — prioritizes the currently-focused task. Noticeably snappier for single-app workloads (gaming, DAWs, video editing). Can slightly penalize heavy background tasks.
- **BMQ** (BitMap Queue) — simple bitmap-based scheduler. Great raw throughput on heavy multi-thread workloads, but niche and **no `sched-ext`** support. Pick only if a specific workload prefers it.

**CachyOS vs vanilla kernels:**
CachyOS kernels add patches for: x86-64-v3/v4 optimized code paths, BBRv3 TCP congestion control, AMD P-State EPP, LZ4 compression in the kernel, and various scheduler/MM tweaks. They live in the **CachyOS repos**, not `[extra]` — so for a CachyOS kernel you must enable the repos on the live ISO ([§3.0a](#30a-optional-enable-cachyos-repos-live-iso)) and select it here in §3.1. There is no post-chroot swap; CachyOS repos are only set up from the live ISO.

**Recommended approach:**
Install `linux-zen` as your daily driver and `linux-lts` as fallback. If you game or do real-time audio, add `linux-cachyos-bore` — picked up front via [§3.0a](#30a-optional-enable-cachyos-repos-live-iso) (CachyOS kernels can't be added later). GRUB picks the first kernel by default; hold Shift during boot to choose another.

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
# KERNEL_PKGS may include a linux-cachyos* variant if you enabled §3.0a on the live ISO.
# pacstrap resolves them from the live ISO's pacman.conf (already carrying the [cachyos-*] repos).
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

# If you enabled CachyOS repos on the live ISO (§3.0a), pacstrap copied pacman.conf
# (it already has the [cachyos-*] sections) but NOT the separate cachyos-*-mirrorlist
# files. Copy them too or pacman in the chroot can't reach the repos:
for m in cachyos-mirrorlist cachyos-v3-mirrorlist cachyos-v4-mirrorlist; do
  [ -f /etc/pacman.d/$m ] && cp /etc/pacman.d/$m /mnt/etc/pacman.d/$m
done
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

> Skip if using Vanilla Arch repos. If you enabled the repos on the live ISO ([§3.0a](#30a-optional-enable-cachyos-repos-live-iso)) you've already done the keyring/mirrorlist/pacman.conf steps — **jump straight to the `cachyos-repo.sh` part below** (it also installs the forked pacman + reinstall). For the vanilla-first route, this whole block adds the repos from scratch.
>
> **Note — the script also installs CachyOS's forked pacman** (`INSTALLED_FROM` tracking + auto arch detection). It's optional: vanilla Arch pacman works fine with the repos. To skip the fork, don't use the script — install just the keyring + mirrorlists by hand and add `cachyos-v3`/`cachyos-v4` to `/etc/pacman.conf` (the old §3.1 block in git history is exactly this).

```bash
# Trust the CachyOS signing key FIRST — without --lsign-key, pacman rejects the
# repo with "signature ... is unknown trust" (the official script usually does this,
# but if the keyserver hiccups you're stuck — doing it manually is idempotent)
sudo pacman-key --recv-keys F3B607488DB35A47 --keyserver keyserver.ubuntu.com
sudo pacman-key --lsign-key F3B607488DB35A47
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

# Replace zlib with CachyOS's zlib-ng-compat (same ABI, faster — avoids
# "Remove zlib?" conflict prompts on every later pacman command).
# --ask=4 answers YES to the conflict/remove prompt (--noconfirm alone answers N).
sudo pacman -S --noconfirm --ask=4 zlib-ng-compat lib32-zlib-ng-compat

# Rank CachyOS mirrors by speed
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
  util-linux inetutils usbutils rsync htop bat zip unzip 7zip \
  avahi nss-mdns \
  alsa-utils sof-firmware easyeffects \
  bluez bluez-utils \
  iwd \
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
systemctl enable iwd
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
> - `sshd` — SSH server (we enabled password auth during install; harden in §9.5)

### 6.3 YAY (AUR Helper)

`yay` is needed for AUR packages in §7 (Noctalia, sddm theme, HyprMod, etc.). Install it now inside chroot so §7 can use it:

```bash
sudo pacman -S --noconfirm --needed git base-devel
cd /tmp
sudo -u pop git clone https://aur.archlinux.org/yay-bin.git
cd yay-bin && sudo -u pop makepkg -si --noconfirm
cd ~ && rm -rf /tmp/yay-bin
```

> We use `yay-bin` (prebuilt) to avoid compiling yay from source. `sudo -u pop` runs the build as your user since `makepkg` refuses to run as root.

---

## §7 — Desktop Stack

> ⚠️ **Pick ONE path.** KDE is all-in-one — install it and stop. Niri needs a shell — install Niri then Noctalia. Hyprland needs a shell — install Hyprland then Noctalia (choose Scrolling or Tiling, not both). Do not mix KDE, Niri, and Hyprland.

Each path has a dedicated companion guide with the full install + config. Open the one you picked:

| Path | Guide | What you get | Login screen |
|------|-------|-------------|--------------|
| 🖥️ **KDE Plasma** | [`KDE_Plasma.md`](KDE_Plasma.md) (+ [theming](KDE_Theming.md)) | Full desktop — compositor, shell, apps, all integrated | `plasma-login-manager` |
| 🏔️ **Niri + Noctalia** | [`Niri_Noctalia_v5.md`](Niri_Noctalia_v5.md) | Scrollable-tiling compositor + native shell (bar, launcher, dock, notifications, wallpaper) | `sddm` + `pixie-sddm-git` |
| 🪟 **Hyprland + Noctalia (Scrolling)** | [`hyprland-Scrolling.md`](hyprland-Scrolling.md) | Tiling compositor with Niri-style scrolling tape + Noctalia shell | `sddm` |
| 🪟 **Hyprland + Noctalia (Tiling)** | [`Hyprland-Tiling.md`](Hyprland-Tiling.md) | Tiling compositor with classic dwindle/master layout + Noctalia shell | `sddm` |
| 💀 **Niri alone** | N/A — hand-pick every component (waybar, fuzzel, swaybg…) | Bare compositor — no bar, no launcher, no wallpaper | none (start from TTY) |

KDE Plasma is the mainstream choice: everything works out of the box, familiar desktop metaphor, KDE apps integrate perfectly. Niri + Noctalia is leaner: scrollable-tiling workflow, lower resource usage, keyboard-driven, but still has a full shell. Hyprland + Noctalia is the tinkerer's pick: Lua config with hot-reload, two layout philosophies, and the same native C++ Noctalia shell.

**No conflict between paths:** each uses its own login manager — KDE → `plasmalogin`, Niri + Hyprland → `sddm` (with the `pixie` theme). Enable exactly one. The KDE packages referenced in the guides (`plasma-integration`, `kded`, `dolphin`) are libraries/apps for KDE apps *on* a compositor — they don't pull the Plasma desktop.

### ▸ Secret Storage (all paths)

Apps need a secrets backend to safely store passwords. GTK apps (VS Code, Chromium, Firefox, Git) use `libsecret`. KDE apps (Dolphin network passwords, KDE Connect) use KWallet. Install both — they coexist fine:

```bash
sudo pacman -S --noconfirm --needed gnome-keyring libsecret kwallet kwalletmanager kwallet-pam
```

> PAM hooks for the login manager are added in the SDDM setup in `Niri_Noctalia_v5.md` — they need `/etc/pam.d/sddm` to exist first.
>
> **KDE path (Plasma Login Manager):** `plasmalogin` ships its own `pam_kwallet`/`pam_gnome_keyring` hooks — verify with `grep -i kwallet /etc/pam.d/plasmalogin`; if missing, add the same `auth optional` / `session optional` lines used for sddm above.
>
> KWallet auto-unlock: wallet password = login password, blowfish encryption, wallet name = `kdewallet`.

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

### 9.2 CachyOS Extras (optional)

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
# more responsive than kernel OOM. Already included in systemd — just enable:
# (no package needed — built into systemd)
# sudo systemctl enable --now systemd-oomd
```

> `cachyos-settings` pulls in `ananicy-cpp` (auto process priority — games/media get higher priority, background tasks lower), `zram-generator` (compressed RAM swap), and CachyOS-specific defaults. `cachyos-gaming-meta` is a convenience bundle — you can also install gaming packages individually in §9.8.

### 9.3 GPU Driver

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

`nvidia-open-dkms` is the open-source kernel module (610.x+). It works with all kernels including linux-zen and linux-cachyos because it builds against your running kernel via DKMS. This whole block runs only when NVIDIA was detected:

```bash
if [ "$gpu_vendor" = "nvidia" ]; then
  # --- 1. Driver + compute ---
  sudo pacman -S --noconfirm --needed \
    nvidia-open-dkms nvidia-utils lib32-nvidia-utils \
    nvidia-settings libxnvctrl \
    ocl-icd opencl-nvidia lib32-opencl-nvidia clinfo cuda

  # --- 2. Gaming extras — VA-API, Vulkan, DXVK ---
  sudo pacman -S --noconfirm --needed \
    libva-utils vdpauinfo vulkan-tools \
    libva-nvidia-driver vkd3d shaderc spirv-tools

  # DXVK is AUR (not in official repos) — needs yay
  # 🔒 AUR — review `yay -G dxvk-bin` before installing
  yay -S --noconfirm --needed dxvk-bin

  # --- 3. Wayland DRM modesetting (kernel cmdline, GRUB) ---
  # For Limine: add nvidia-drm.modeset=1 nvidia-drm.fbdev=1 to the CMDLINE: line instead.
  if grep -q 'GRUB_CMDLINE_LINUX_DEFAULT' /etc/default/grub 2>/dev/null; then
    sudo sed -i '/^GRUB_CMDLINE_LINUX_DEFAULT=/s/"$/ nvidia-drm.modeset=1 nvidia-drm.fbdev=1"/' /etc/default/grub
    sudo grub-mkconfig -o /boot/grub/grub.cfg
  fi

  # --- 4. mkinitcpio: add NVIDIA modules, remove the generic kms hook (NVIDIA provides its own) ---
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

  # --- 5. Pacman hook: auto-rebuild initramfs on driver/kernel updates ---
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

### 9.4 Snapper

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

### 9.5 SSH Hardening

> ⚠️ **Do this after setting up SSH keys** (§9.1 client keygen + `ssh-copy-id`). Otherwise you'll lock yourself out.

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

### 9.6 Firewall

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

### 9.7 AppArmor (optional)

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

### 9.8 Extra Packages & Fonts

Personal pick list — install what you need:

```bash
# Core CLI + media
sudo pacman -S --noconfirm --needed imagemagick gimp obs-studio \
  vlc vlc-plugin-ffmpeg vlc-plugin-mpeg2 vlc-plugin-x264 vlc-plugin-x265 \
  vlc-plugin-ass vlc-plugin-matroska vlc-plugin-dvd vlc-plugin-bluray \
  vlc-plugin-srt vlc-plugin-soxr libdvdcss libbluray

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

# Custom Nerd Font — Thai + Nerd Icons (Ghostty terminal)
# Download: https://github.com/poppatchara/NotoSansMThaiNerdFont
# Install: cp -r NotoSansMThaiNerdFont/ ~/.local/share/fonts/ && fc-cache -f
sudo pacman -S --noconfirm --needed \
  noto-fonts noto-fonts-emoji \
  ttf-dejavu ttf-ubuntu-font-family \
  terminus-font ttf-nerd-fonts-symbols

# Microsoft fonts (AUR)
yay -S --noconfirm --needed ttf-ms-fonts

# Spell check
sudo pacman -S --noconfirm --needed hunspell hunspell-en_us hunspell-en_gb
yay -S --noconfirm --needed hunspell-th  # Thai dictionary

# Flatpak (optional — for sandboxed apps)
sudo pacman -S --noconfirm --needed flatpak
```

> Run this from the logged-in session (not SSH):

```bash
flatpak install -y flathub \
  io.github.jonmagon.kdiskmark \
  io.github.flattool.Warehouse \
  io.missioncenter.MissionCenter \
  xyz.z3ntu.razergenie \
  io.github.arunsivaramanneo.GPUViewer \
  com.kgurgul.cpuinfo \
  org.raspberrypi.rpi-imager \
  io.github.shonubot.Spruce \
  org.localsend.localsend_app
```

### 9.9 pyenv

`pyenv` manages multiple Python versions per-user without conflicting with the system Python:

```bash
# zlib vs zlib-ng-compat: CachyOS replaces zlib upstream. Install whichever
# this system already uses, so the two never conflict (requesting `zlib`
# while zlib-ng-compat is installed would make pacman ask to remove it).
zlib_pkg="zlib"
pacman -Q zlib-ng-compat >/dev/null 2>&1 && zlib_pkg="zlib-ng-compat"
lib32_zlib_pkg="lib32-zlib"
pacman -Q lib32-zlib-ng-compat >/dev/null 2>&1 && lib32_zlib_pkg="lib32-zlib-ng-compat"
sudo pacman -S --noconfirm --needed openssl "$zlib_pkg" "$lib32_zlib_pkg" xz tk readline sqlite libffi bzip2
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

> **Why the zlib guard?** CachyOS replaces `zlib` with `zlib-ng-compat` (same ABI, faster, done in the CachyOS Repos section). The lines above detect which one this system uses and install it — so the command works on **both** vanilla Arch and CachyOS paths without pacman asking to remove `zlib-ng-compat`. pyenv compiles fine against either. `--needed` skips anything already installed.

### 9.10 SPDIF Audio Fix (optional)

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

### 9.11 Cache Cleanup

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
