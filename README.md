# Arch Linux Install Memo 🐧

Personal notes for rebuilding my daily Arch install: UEFI, single NVMe, Btrfs root.
Not the best way. Just the way I like.

---

## 📖 The Guide

| Guide | Description |
|-------|-------------|
| [**Arch Linux Installation Guide**](Arch%20Linux_Installation_Guide.md) ⭐ | **Unified guide** — all decisions in one file |
| [Niri + Noctalia v5](Niri_Noctalia_v5.md) 🏔️ | Companion — detailed Niri config, keybinds, greeter reference |

## 🧱 Decision Matrix

| Decision | A | B | C |
|----------|---|---|---|
| **Kernel** | linux-zen | linux-cachyos | linux / linux-lts |
| **Repos** | Vanilla Arch | CachyOS | |
| **Desktop** | KDE Plasma | Niri + Noctalia | |
| **Bootloader** | GRUB | Limine | |

All 4 decisions are independent. See the unified guide for detailed walkthrough.

---

## 📝 Changelog

### 2026-07-19

- **Unified rewrite:** Merged all individual guides into one unified guide with decision matrix.
- **Niri + Noctalia split:** Compositor (§7.2) and shell (§7.3) are now separate sections — ready for future compositors (Hyprland, MangoWC, DMS).
- **Kernel independence:** Kernel choice is now a free variable — `linux-zen`, `linux-cachyos`, `linux`, `linux-lts` — independent of repo choice.
- **UUID auto-detection:** All partition UUIDs now detected by type (ESP GUID / swap / btrfs), no hardcoded `/dev/nvme0n1pN` paths.
- **Explanations:** Added context, rationale, and "why" throughout every section.
- **Archived:** Old individual guides moved to `archive/`. Fedora guide removed.

### 2026-07-13

- **GRUB + CachyOS:** Switched from Limine to GRUB for CachyOS combo — `/boot` on Btrfs `@boot` subvolume for snapshots. *(Zen_Grub_CachyOS)*
- **Niri + Noctalia (fresh):** Initial version — forked from Zen_Grub_Brtfs. Replaced KDE Plasma with Niri + Noctalia v5. Uses `greetd` + `noctalia-greeter` instead of `plasma-login-manager`. *(Zen_Niri_Noctalia)*

### 2026-06-16

- **AUR security hardening:** Following the [June 2026 AUR malware incident](https://archlinux.org/news/active-aur-malicious-packages-incident/), added PKGBUILD review checklist. Moved all official-repo packages from `yay` to `pacman`. AUR-only packages marked 🔒. *(Zen_Grub_Brtfs)*
- **Cleanup:** Removed unreferenced packages (`inotify-tools`, `openbsd-netcat`, `vim`, `wget`, `snapper-tools`, `acpi_call`). Consolidated 8 separate `pacman -S` calls into one. *(Zen_Grub_Brtfs)*
- **SSH fixes:** Removed deprecated `Protocol 2`, replaced `ChallengeResponseAuthentication` with `KbdInteractiveAuthentication`, removed insecure `ForwardAgent yes`. *(Zen_Grub_Brtfs)*

### 2026-06-07

- **Plasma Login Manager PAM:** Fixed critical login failure — PLM doesn't ship a default PAM config. Added mandatory `/etc/pam.d/plasmalogin` creation step. *(Zen_Grub_Brtfs, Zen_Grub_CachyOS)*
- **GPU auto-detection:** `lspci`-based GPU vendor detection. NVIDIA driver only installed if dGPU present. *(all guides)*
- **NVIDIA open driver:** Replaced proprietary option with single `nvidia-open-dkms` (610.x). *(all guides)*
- **pyenv:** Bumped Python install from 3.12 → 3.13.2. *(all guides)*

### 2026-05-16

- **SDDM → Plasma Login Manager:** Replaced SDDM with `plasma-login-manager` (Plasma 6.6+ native login manager). *(Zen_Grub_Brtfs)*
- **NVIDIA 590xx:** Updated AUR driver reference (580xx no longer exists). *(Zen_Grub_Brtfs)*

### 2026-03-21

- **GRUB variant created:** Forked from Limine guide because Limine cannot boot from Btrfs `/boot`. Wanted `/boot` in snapshots. *(Zen_Grub_Brtfs, Zen_Grub_CachyOS)*

### 2025-12-26

- **KDE theme stutter:** Narrowed down to specific themes — Whitesur, Orchis, Colloid cause stutter. Qogir, Darkly, Vinyl are safe. *(Zen_Grub_CachyOS)*

### 2025-12-22

- **linux-zen over CachyOS:** Switched back to `linux-zen` due to minor KDE Plasma window resize stutter with CachyOS kernels. *(Zen_Grub_CachyOS)*

---

## 📁 Archive

Old individual guides (merged into unified guide above):

- [`archive/Arch Linux_Zen_Grub_Brtfs.md`](archive/Arch%20Linux_Zen_Grub_Brtfs.md) — GRUB + linux-zen + KDE
- [`archive/Arch Linux_Zen_Grub_CachyOS.md`](archive/Arch%20Linux_Zen_Grub_CachyOS.md) — GRUB + CachyOS + KDE
- [`archive/Arch Linux_Zen_Niri_Noctalia.md`](archive/Arch%20Linux_Zen_Niri_Noctalia.md) — GRUB + linux-zen + Niri+Noctalia

---

## 🙏 Credits & Thanks

Huge thanks to the authors/maintainers of these guides and notes that helped shape parts of this memo:

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

---

*Last updated: July 2026*
