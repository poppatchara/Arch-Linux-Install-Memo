# Arch Linux Install Memo 🐧

Personal notes for rebuilding my daily Arch install: UEFI, single NVMe, Btrfs root.
Not the best way. Just the way I like.

---

## 📖 The Guide

| Guide | Description |
|-------|-------------|
| [**Arch Linux Installation Guide**](Arch%20Linux_Installation_Guide.md) ⭐ | **Unified guide** — all decisions in one file; §7 links to each desktop guide |
| [KDE Plasma](KDE_Plasma.md) 🖥️ | Companion — KDE install, mandatory PAM config, apps, KWallet (+ [theming](KDE_Theming.md)) |
| [Niri + Noctalia v5](Niri_Noctalia_v5.md) 🏔️ | Companion — detailed Niri config, keybinds, greeter reference |
| [Hyprland + Celestia](hyprland-Scrolling.md) 🪟 | Companion — Hyprland scrolling layout, Lua config, Celestia shell, SDDM |
| [Hyprland + Celestia — Native Tiling](Hyprland-Tiling.md) 🪟 | Companion — Hyprland dwindle/master tiling, Lua config, Celestia shell, SDDM |

## 🧱 Decision Matrix

| Decision | A | B | C |
|----------|---|---|---|
| **Kernel** | linux-zen | linux-cachyos | linux / linux-lts |
| **Repos** | Vanilla Arch | CachyOS | |
| **Desktop** | KDE Plasma | Niri + Noctalia | Hyprland + Celestia *(scrolling or tiling)* |
| **Bootloader** | GRUB | Limine | |

All 4 decisions are independent. See the unified guide for detailed walkthrough.

---

## 📝 Changelog

### 2026-08-16

- **§3.0a CachyOS repos now inserted ABOVE `[core]` (fix for vanilla-pacman order bug):** `Arch Linux_Installation_Guide.md`'s live-ISO repo setup previously did `tee -a` (append at the end of `pacman.conf`), which put `[cachyos*]` *last* — so `[core]` won and pacstrap pulled vanilla Arch pacman. That vanilla libalpm doesn't know CachyOS's `%INSTALLED_DB%` package metadata field, producing an "unknown key '%INSTALLED_DB%'" warning flood on every `yay`/`pacman -Qi`. Replaced with a `sed -i '/^\[core\]/i ...'` that inserts the three CachyOS repos *before* `[core]`, so the forked pacman + optimization win (matching how Orizon ended up with the fork). Also added a verify `grep` and a note in the chroot [CachyOS Repos] section telling vanilla-first users to place repos above `[core]` too.
- **Swap is now a file on a `@swap` subvolume (no swap partition):** `Arch Linux_Installation_Guide.md` drops the swap partition entirely for both GRUB and Limine — every layout is just the ESP plus one Btrfs partition. Swap moves to a dedicated `@swap` subvolume (required because Btrfs forbids snapshots of a subvolume holding an active swap file). §1 removes `mkswap`/swap UUID detection; §2.1 adds `@swap`; §2.2 mounts it and creates the file with `btrfs filesystem mkswapfile`; §2.4 fstab gains the `/swap/swapfile` line (genfstab doesn't write swap files); §5.1 re-detects `resume_uuid` + `resume_offset` (via `btrfs inspect-internal map-swapfile`, not `filefrag`) and both bootloaders pass `resume=UUID=... resume_offset=...` instead of `resume=UUID=<swap partition>`.
- **`/boot` moved into `@` (no separate `@boot`):** `Arch Linux_Installation_Guide.md` no longer creates a `@boot` subvolume. `/boot` now lives inside `@`, so a single `@` snapshot covers kernel + initramfs + system — rollback is all-or-nothing and simpler. This cascaded through §2.2 (drop `@boot` create, mount `@` covering `/boot`), §2.3 (GRUB only mounts the ESP at `/boot/EFI` inside Btrfs `/boot`; Limine mounts the ESP over `/boot`), and §9.4 Snapper (removed the separate `boot` snapper config — `root` covers it). §1 decision text updated to match.
- **Refactor §5 — merged GRUB/Limine bootloader prep:** `Arch Linux_Installation_Guide.md` now has **▸ 5.1 Shared Bootloader Prep** (microcode + swap/root UUID detection, the shared kernel-parameter core, and the identical zswap `MODULES` + `mkinitcpio -P` block) followed by **▸ 5.2 GRUB** and **▸ 5.3 Limine** that only keep what differs — install/deploy mechanics, kernel-command-line application (GRUB omits `root=` since its search hook finds `/`; Limine inlines `root=UUID=... rootflags=subvol=@`), config generation, and post-update hooks. Removes the ~20-line duplicated zswap/mkinitcpio + microcode/UUID-detection blocks.
- **§3.1 kernel options expanded:** `Arch Linux_Installation_Guide.md`'s `KERNELS` block + kernel reference now cover the full set of popular kernels — vanilla `linux-hardened` (security) and `linux-rt` (real-time, PREEMPT_RT) added alongside `linux-zen`/`linux`/`linux-lts`; CachyOS side now lists `linux-cachyos` (default), `-bore`, `-eevdf`, `-bmq`, `-lts`, `-deckify`, `-hardened`. Scheduler explainer gained a **BMQ** entry. Table split into vanilla vs CachyOS groups with repo/scheduler columns verified against the live repos.
- **Removed post-chroot CachyOS kernel swap:** the `Optional: swap to a CachyOS kernel` section in CachyOS Repos is gone, and every "swap kernel post-chroot" reference (Decision Matrix rows 1/2 + note, §3 intro bullets + Repos decision, §3.0a intro, §3.1 note/table) now states the only way to get a `linux-cachyos*` kernel is up front enable §3.0a on the live ISO. CachyOS repos post-chroot still provide optimized binaries for non-kernel packages.
- **§3.0a mirrorlist install is now version-dynamic:** the hardcoded `pacman -U https://mirror.cachyos.org/.../cachyos-keyring-20240331-1-...` pins are replaced with a small loop that resolves the latest package from the live repo directory index (`curl … | sort -V | tail -1`) — no hardcoded version to go stale when CachyOS bumps a mirrorlist package. CachyOS has no `-latest` symlink, so dirindex resolution is the reliable path.
- **CachyOS-first route — enable CachyOS repos on the live ISO:** `Arch Linux_Installation_Guide.md` gains **▸ 3.0a Optional: Enable CachyOS Repos (Live ISO)** — a manual keyring + mirrorlist + `pacman.conf` setup that lets §3.1 select a `linux-cachyos*` kernel and §3.3 pacstrap it directly (no double-install). The `cachyos-repo.sh` fork/reinstall stays in the chroot [CachyOS Repos] section as before. Decision Matrix rows 1/2 updated with both routes; §3.3 gains the critical `cp` of the separate `cachyos-*-mirrorlist` files into `/mnt` (pacstrap only copies the Arch mirrorlist + pacman.conf). TOC + all internal links updated. Known pitfall preserved: on the live ISO, touch only the repos — the shell script is a chroot operation.
- **Refactor §2.2/§2.3 — merged GRUB & Limine mounts:** `Arch Linux_Installation_Guide.md` now has **▸ 2.2 Common Mounts** (shared block: creates both `@boot` + `@srv`, mounts every subvolume except boot/ESP, swap) plus **▸ 2.3 Boot & ESP Mount (per-bootloader)** splitting only where they diverge — GRUB (`@boot` Btrfs mount + ESP at `/boot/EFI`) vs Limine (ESP directly at `/boot`). Removes the ~10-line duplicate mount sequence, keeps the boot/ESP-only difference explicit, and fixes a stale `§5.2` cross-reference.

### 2026-08-15

- **Niri now uses SDDM + pixie (replaces greetd/noctalia-greeter):** per user decision — Niri should use the same login manager as Hyprland. `Niri_Noctalia_v5.md`'s `Switch to Noctalia Greeter` section replaced with an **SDDM Login Manager** section (`sddm` + `pixie-sddm-git`, Qt6 prereqs, `/etc/sddm.conf.d/theme.conf` `Current=pixie`), PAM hooks moved to `/etc/pam.d/sddm`, cursor theme switched `capitaine-cursors` → **Phinger** (`phinger-cursors`) + **Bibata Classic** alternative (matches the Hyprland guide). Decision matrix updated in `Arch Linux_Installation_Guide.md`: Niri + Hyprland → `sddm`, KDE → `plasmalogin`. Historical `greetd` notes kept in the changelog/archive.
- **`niri-qol-git` optional option:** Added to `Niri_Noctalia_v5.md` Installation — soft-fork of Niri that fast-tracks 3 upstream QoL PRs (sticky floating windows #3302, hidden workspaces #2997, `float-above-fullscreen` #4062). Replaces base `niri`; safety-checked (clean PKGBUILD, `--locked`/`--frozen` build, no network hooks) but 0-vote brand-new package → marked experimental. Step 1 install cross-referenced to point to the alternative.

### 2026-08-13

- **Main guide §7 now covers Hyprland:** `Arch Linux_Installation_Guide.md` gains a **▸ Hyprland + Celestia** section — package matrix (which packages each route installs/drops), shared base install, Route A (scrolling + ScrollOverview) vs Route B (native tiling, no plugin), SDDM + pixie theme setup, and pointers to the two companion guides. Decision Matrix + Recommended Combos updated; Secret Storage promoted to shared (all paths) with a SDDM/KWallet note. No conflicts: KDE→plasmalogin, Niri+Hyprland→sddm.
- **Native Tiling companion added:** New `Hyprland-Tiling.md` — same stack as `hyprland-Scrolling.md` but classic tiling: **dwindle** default (+ tweaks), **master** as optional per-workspace layout, `hjkl`/arrow keybinds (focus/move/resize), `SUPER+Tab` cycle windows, no ScrollOverview/plugin. Cursor themes (Phinger/Bibata), pixie-sddm, and live wallpapers (mpvpaper) included.
- **pixie-sddm theme:** SDDM section now recommends pixie-sddm (Material Design 3 / Pixel UI, AUR `pixie-sddm-git`) with Qt6 prereqs + theme.conf setup.
- **Cursor themes:** New "Cursor Themes" section — Phinger (XCursor) + Bibata Classic (`bibata-cursor-git`, hyprcursor + XCursor), `hyprctl setcursor` for native, `XCURSOR_THEME`/`XCURSOR_SIZE` env for XWayland/GTK.
- **Live wallpapers (optional):** Celestia section gains mpvpaper how-to — MotionBGs `.mp4` download, `mpvpaper -o loop`, autostart in Lua, layer-shell conflict note with Celestia's static-only wallpaper.

### 2026-08-12

- **Hyprland + Celestia companion added:** New `hyprland-Scrolling.md` — native scrolling layout (0.55+), Lua config (`hyprland.lua`), Celestia shell (Quickshell-based: bar/launcher/notif/lock/idle/paper/picker in one), SDDM login manager, XWayland notes. Third Desktop option in the decision matrix.
- **ScrollOverview plugin:** Added §5 to `hyprland-Scrolling.md` — Niri-style overview (zoom-out all workspaces, trackpad swipe, ALT+Tab visual switcher) via `hyprpm`, with Lua config, submap, gestures, and dispatchers.
- **Native screenshot stack:** Added `grim` + `slurp` + `satty` + `wl-clipboard` to `hyprland-Scrolling.md` — Print/SUPER+Print/CTRL+Print binds for region-annotate, fullscreen-annotate, and clipboard copy. No portal needed.
- **HyprMod GUI settings:** Added §6 to `hyprland-Scrolling.md` — live-preview settings app, `SUPER+M` opens the GUI. Settings-only (no layout-switch profile cycling).
- **Dolphin file manager:** Added to `hyprland-Scrolling.md` — `SUPER+E` opens Dolphin (Hyprland convention; "next empty workspace" moved to `SUPER+N`), full KDE Apps on Hyprland section (plasma-integration + kded6, KWallet-only secret storage via Secret Service interface — no GNOME Keyring, "Open With" fix, dual env via `environment.d` + `hl.env()`).
- **snappy-switcher (optional):** Added §5.6 to `hyprland-Scrolling.md` — big per-window thumbnail ALT+Tab alternative to ScrollOverview's built-in switcher, marked optional (default stays ScrollOverview ALT+Tab).
- **Walker launcher:** Added §8 to `hyprland-Scrolling.md` — Wayland-native launcher (GTK4/Rust) with elephant backend, `SUPER+Space` now opens Walker (replaces Celestia's built-in launch), providers table (calc/files/runner/clipboard/symbols), TOML config example.
- **Niri-style keybinds:** Rewrote §4.2 `keybinds.lua` to mirror Niri muscle memory — `SUPER+T` terminal, HJKL+arrows focus, `SUPER+Home/End` first/last column, `SUPER+Ctrl+H/L` move column, `SUPER+F` maximize / `SUPER+Shift+F` fullscreen, `SUPER+C` center, `SUPER+Shift+S` region screenshot, `Super+Alt+L` lock, `SUPER+Shift+E` logout.
- **Niri-style wheel + workspace binds:** Added to §4.2 — `SUPER+scroll` column focus, `SUPER+Shift+scroll` workspace switch, `SUPER+U/I` prev/next workspace, `SUPER+Shift+U/I` move window to prev/next workspace.
- **Niri → Hyprland shortcut map:** Added table to §4.2 — full Niri-vs-Hyprland keybind comparison (verified against real Niri config), plus a "not mapped" list (monitor nav, width ±10%, tabbed, etc.).
- **IPC explainer:** Added "What IPC is" + verification note to the Celestia section.
- **PCoIP section rewritten:** §9 now reflects Hyprland's integrated XWayland (no xwayland-satellite) — native Wayland path via pcoip-client fork + XWayland fallback, no config-swap hack; keyboard-inhibit levers (`dont_inhibit`, `no_shortcuts_inhibit`) + test checklist.
- **pixie-sddm theme:** SDDM section now recommends pixie-sddm (Material Design 3 / Pixel UI, AUR `pixie-sddm-git`) with Qt6 prereqs + theme.conf setup.
- **Cursor themes:** New "Cursor Themes" section — Phinger (XCursor) + Bibata Classic (`bibata-cursor-git`, hyprcursor + XCursor), `hyprctl setcursor` for native, `XCURSOR_THEME`/`XCURSOR_SIZE` env for XWayland/GTK.
- **Live wallpapers (optional):** Celestia section gains mpvpaper how-to — MotionBGs `.mp4` download, `mpvpaper -o loop`, autostart in Lua, layer-shell conflict note with Celestia's static-only wallpaper.

### 2026-07-19

- **Unified rewrite:** Merged all individual guides into one unified guide with decision matrix.
- **Niri + Noctalia split:** Compositor (§7.2) and shell (§7.3) are now separate sections — ready for future compositors (Hyprland, MangoWC, DMS).
- **Kernel independence:** Kernel choice is a free variable — `linux-zen`, `linux`, `linux-lts` work on Vanilla Arch; **any `linux-cachyos*` kernel requires the CachyOS repos** (added post-chroot, then the kernel is swapped in the CachyOS Repos section).
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
