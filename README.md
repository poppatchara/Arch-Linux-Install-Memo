# Arch Linux Install Memo 🐧

Personal notes for rebuilding my daily Arch install: UEFI firmware, single NVMe drive, Btrfs root, KDE Plasma desktop.

*Not the best way or most correct way. Just the way I like.*

---

## 📖 Guides

| Guide | Bootloader | Kernel | Notes |
|-------|------------|--------|-------|
| [**Arch Linux + GRUB + Btrfs**](Arch%20Linux_Zen_Grub_Brtfs.md) ⭐ | GRUB | linux-zen | **Main guide** — includes Snapper, KDE Plasma 6.6+ (Plasma Login Manager), SSH hardening, pyenv |
| [**Arch Linux + GRUB + CachyOS**](Arch%20Linux_Zen_Grub_CachyOS.md) | GRUB | CachyOS | CachyOS kernels, KDE Plasma, GRUB bootloader, Snapper snapshot integration |
| [**Fedora 44 KDE (Lean)**](Fedora%20KDE_Lean.md) | GRUB (default) | Fedora kernel | **Fedora variant** — minimal KDE, dnf5, RPM Fusion for NVIDIA, Btrfs by default |
|| [**Niri + Noctalia v5**](Niri_Noctalia_v5.md) 🏔️🆕 | — | linux-zen | **Companion guide** — config, keybinds, tweaks for Niri+Noctalia on existing Plasma setup |
|| [**Arch + Niri + Noctalia v5 (Fresh)**](Arch%20Linux_Zen_Niri_Noctalia.md) 🏔️⭐ | GRUB | linux-zen | **New!** Full fresh install — Niri+Noctalia instead of KDE Plasma, lean but feature-rich |

---

## 🧱 Base Setup (All Guides)
- **Partition:** ESP + Btrfs root + swap
- **Filesystem:** Btrfs with subvolumes (`@`, `@home`, `@var`, `@var_log`, `@var_cache`, `@root`, `@srv`)
- **Desktop:** KDE Plasma 6.6+ with Plasma Login Manager
- **GPU:** Auto-detect vendor (NVIDIA/Intel/AMD) — install only if dGPU present

---

*Last updated: June 2026*
