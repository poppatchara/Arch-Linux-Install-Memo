# AGENTS.md

Guidance for AI coding agents working in this repository.

## Repo Overview

Personal notes for rebuilding a daily Arch Linux install: UEFI, single NVMe, Btrfs root.
Not the best way. Just the way the owner likes it.

- **Unified guide**: `Arch Linux_Installation_Guide.md` — all decisions in one file (§0–§8)
- **Companion files**: one per desktop/compositor stack (e.g. `Niri_Noctalia_v5.md`, future `hyprland-Scrolling.md`)
- **Decision matrix**: in `README.md` — every setup choice (kernel, repos, desktop, bootloader) is a free variable decided there

## Repository Layout

| Path | Purpose |
|------|---------|
| `README.md` | Entry point, decision matrix, changelog |
| `Arch Linux_Installation_Guide.md` | The unified guide — main content |
| `Niri_Noctalia_v5.md` | Companion: Niri + Noctalia v5 stack |
| `PCoIP_Niri_Workaround.md` | Companion: PCoIP keyboard passthrough on Niri |
| `KDE_Theming.md` | Companion: KDE theme notes |
| `.markdownlint.json` | Markdown lint rules |
| `archive/` | Old individual guides — **read-only reference, do not edit** |

## Editing Conventions

1. **Language**: guides are written in English (shell comments may use English).
2. **Decision matrix is law**: never silently change kernel/repo/desktop/bootloader choices.
   If a section depends on a matrix choice, state which one it assumes.
3. **Explain the "why"**: every command block gets a comment or note explaining why it exists.
4. **Shell blocks**:
   - Use `bash` fenced code blocks.
   - Commands are runnable as-is from a fresh install or chroot (no placeholder assumptions
     unless clearly marked).
   - User-specific paths (e.g. `/home/pop/...`) are allowed only when the guide is explicitly
     personal; prefer generic paths (`/home/<user>/...` or `$HOME`) for reusable instructions.
   - Sensitive values (API keys, tokens) must be placeholders only — never real secrets.
5. **Markdown**: must pass `.markdownlint.json` rules (MD033 allows `details`, `summary`, `br`).
   Run `markdownlint-cli2 "**/*.md"` if available.
6. **Sections**: the guide uses `§` numbering (`§0` … `§8`). Keep new content in the right section;
   companion files get their own top-level heading matching the pattern of existing companions.

## Workflow

- Make focused changes per commit.
- Conventional commit messages: `feat:`, `doc:`, `fix:`, `refactor:`.
- Do **not** edit `archive/` — those are merged-in old guides kept for reference.
- When adding a new companion file, update `README.md` table + changelog in the same commit.
- Push to `origin/main` when done; the repo is mirrored across Orizon and Chuwi machines.

## Verification

- `git status` clean after your changes (no stray files).
- Markdown lint passes.
- Code blocks are syntactically valid bash (spot-check with `bash -n` when in doubt).
