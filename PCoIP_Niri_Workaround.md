# PCoIP Remote Desktop — Niri Keyboard Passthrough Workaround

pcoip-client (HP Anyware) runs natively on Wayland under Niri but does **not** capture keyboard shortcuts (Mod/Super, Alt+Tab, Mod+drag) because Niri intercepts them at the compositor level.

> **Prerequisite:** the `pcoip-client` package must be built with the native Wayland fix (vendored Qt 6.9 Wayland client + kept Wayland platform plugins). See the [pcoip-client repo](https://github.com/pop/pcoip-client).

---

## Problem

PCoIP client captures keyboard/mouse input for remote desktop sessions, but Niri steals `Mod+` shortcuts and mouse drags before they reach the client:

| Layer | Problem | Effect |
|-------|---------|--------|
| Niri `binds {}` | 110+ `Mod+` binds intercept Super-key combos | Remote desktop can't receive Super+C, Super+V, etc. |
| Niri `recent-windows` | Built-in Alt+Tab switcher (since v25.11) | Alt+Tab switches local windows instead of forwarding to remote |
| Niri default gestures | Mod+MouseLeft (drag), Mod+MouseRight (resize) | Mouse drags move Niri windows instead of dragging in remote session |
| nirimod | Auto-replaces config, strips binds | Config gets overwritten mid-session |

---

## Solution: Dual Config Auto-Swap

A wrapper script at `~/.local/bin/pcoip` manages two Niri config files:

- **config-normal.kdl** — standard keybinds, used when PCoIP is not focused
- **config-pcoip.kdl** — all `Mod+` binds commented out, used when PCoIP has focus

The wrapper auto-detects focus transitions via `niri msg -j event-stream` and swaps config on the fly. Niri auto-reloads config within ~100ms.

### Flow

```
Launch pcoip → kill nirimod → failsafe check → start focus watcher
  ↓
pcoip-client gains focus → save config.kdl → swap to config-pcoip.kdl
  ↓
pcoip-client loses focus → restore config-normal.kdl
  ↓
Exit pcoip → cleanup restores config-normal.kdl
```

### Why This Works

Niri emits `WindowFocusChanged` events via IPC. The wrapper watches these events and swaps config files on focus transitions. The `NIRI_SOCKET` export is critical — launchers (Noctalia, wofi, desktop entries) often don't set this variable.

### What Was Tried and Failed

| Approach | Issue |
|----------|-------|
| gamescope wrapper | Alt+Tab still intercepted by host compositor, small resolution |
| `toggle-keyboard-shortcuts-inhibit` | Requires app to support `zwp_keyboard_shortcuts_inhibit_manager_v1` — pcoip-client does not |
| Manual config swap on launch | Mod keys disabled systemwide, not per-window |
| nirimod auto-config | Strips `binds {}` section, overwrites manual changes |

---

## nirimod Compatibility

nirimod is a GTK4 GUI for editing Niri config. It can coexist with PCoIP, but must be stopped before launching PCoIP (handled automatically by the wrapper).

**How nirimod works:**
- On first run, it backs up your config to `~/.config/nirimod/baseline/`
- It manages `config.kdl` directly — reads on open, writes on Save (Ctrl+S)
- Binds are preserved: if you add binds via the GUI and Save, they are written to `config.kdl`
- Its `auto_update` setting means it may write to `config.kdl` at any time — hence we stop it

**Workflow:**
1. Use nirimod normally — edit binds, layout, etc. via GUI → Save
2. Launch `pcoip` → wrapper auto-kills nirimod (`pkill -f nirimod`)
3. PCoIP runs with focus-based config swap
4. Exit PCoIP → normal config restored
5. Re-open nirimod when needed — it reads current config.kdl (including all binds)

---

## config-normal.kdl

Standard Niri keybinds — identical to your working `config.kdl`, with permanent no-op overrides for Mod+Mouse:

```kdl
// PCoIP compatibility: Mod+MouseLeft/Right are permanent no-ops
// so mouse drags never move/resize Niri windows.
// These stay in BOTH configs.

binds {
    Mod+MouseLeft  { spawn "true"; }
    Mod+MouseRight { spawn "true"; }
    // ... all other standard binds ...
}
```

The permanent no-ops for `Mod+MouseLeft` and `Mod+MouseRight` live in **both** configs so that even when PCoIP is not focused, dragging with Super held down never moves/resizes Niri windows. This is safe because no standard workflow relies on Mod+Left-click to move windows.

---

## config-pcoip.kdl

Identical to `config-normal.kdl` except:

### 1. All Mod+ binds commented out

Every `Mod+` keybind is commented out so Niri doesn't intercept any keyboard shortcuts. The wrapper generates this from `config-normal.kdl`:

```bash
cp ~/.config/niri/config-normal.kdl ~/.config/niri/config-pcoip.kdl

sed -E '
/^    Mod\+/{
  /Mod\+Escape allow-inhibiting/b
  /Mod\+Tab /b
  /Mod\+F /b
  /Mod\+MouseLeft/b
  /Mod\+MouseRight/b
  /Mod\+Shift\+S /b
  /Mod\+Shift\+V /b
  /Mod\+Alt\+[JK] /b
  /Mod\+Alt\+Up /b
  /Mod\+Alt\+Down /b
  /Mod\+.*[Ww]heel[Ss]croll/b
  s/^    /    \/\//
}' ~/.config/niri/config-normal.kdl > ~/.config/niri/config-pcoip.kdl
```

### 2. recent-windows disabled

Appended to `config-pcoip.kdl`:

```kdl
recent-windows { off; }
```

This lets Alt+Tab pass through to the remote machine instead of triggering Niri's built-in window switcher.

### 3. Mod+MouseLeft/Right bound to no-op

Already present in both configs:

```kdl
Mod+MouseLeft  { spawn "true"; }
Mod+MouseRight { spawn "true"; }
```

### 4. All other settings identical to normal

Layout, appearance, animations, window rules — everything else is preserved unchanged.

### Whitelisted Binds (still active in PCoIP mode)

These binds are preserved (not commented out) in `config-pcoip.kdl`:

| Bind | Action | Why kept |
|------|--------|----------|
| `Mod+Escape` | Toggle keyboard shortcuts inhibit | Escape hatch — always needed |
| `Mod+Tab` | Toggle overview | Quick local workspace view |
| `Mod+F` | Maximize column | Useful for repositioning |
| `Mod+Shift+S` | Region screenshot (Noctalia) | Local screenshot still needed |
| `Mod+Shift+V` | Clipboard (Noctalia) | Local clipboard access |
| `Mod+Alt+J/K/Up/Down` | Workspace switch | Navigate without Alt+Tab |
| `Mod+WheelScroll*` (12 binds) | Workspace/column scroll | Scroll navigation |

---

## Qt Compatibility

The pcoip-client package bundles Qt 6.9.3 and now vendors matching Qt 6.9 `libQt6WaylandClient`/`libQt6WaylandEglClientHwIntegration` libs, keeping the upstream Wayland Qt platform plugins. The package wrapper prefers native Wayland when `WAYLAND_DISPLAY` is set and falls back to X11/XCB, so no manual `QT_QPA_PLATFORM` tweaking is needed.

---

## Launcher Integration

Override the system desktop entry so Noctalia/KDE launchers use the wrapper instead of the raw binary.

`~/.local/share/applications/pcoip-client.desktop`:

```desktop
[Desktop Entry]
Type=Application
Version=1.0
Name=PCoIP (Wrapped)
Comment=Remote access client with Niri config swap
Exec=/home/pop/.local/bin/pcoip
Icon=pcoip-client
Terminal=false
Categories=Network;RemoteAccess;
```

> Use absolute path — `~/.local/bin` may not be in launcher `$PATH`.

---

## Wrapper Script

Full `~/.local/bin/pcoip`:

```bash
#!/usr/bin/env bash
# PCoIP client with focus-based config swap.
# Auto-detects Niri socket and swaps config when pcoip-client gains/loses focus.

# Stop nirimod to prevent it from overwriting config during PCoIP
pkill -f nirimod 2>/dev/null

# Auto-detect Niri socket (required for niri msg to work)
export NIRI_SOCKET=$(ls /run/user/1000/niri.*.sock 2>/dev/null | head -1)

NIRI_CONFIG="$HOME/.config/niri/config.kdl"
NORMAL="$HOME/.config/niri/config-normal.kdl"
PCOIP="$HOME/.config/niri/config-pcoip.kdl"
STATE_FILE="/tmp/pcoip-focus-state"

# Failsafe: if pcoip config is active but pcoip-client isn't focused, restore normal.
if [ "$(cat "$STATE_FILE" 2>/dev/null)" = "ON" ]; then
    FOCUSED_APP=$(niri msg -j focused-window 2>/dev/null | grep -oP '"app_id":\s*"\K[^"]*')
    if [ "$FOCUSED_APP" != "pcoip-client" ]; then
        cp "$NORMAL" "$NIRI_CONFIG"
        echo "[pcoip] Failsafe: restored normal config (stale pcoip state)"
    fi
fi

focus_watch() {
    local CURRENT="normal"

    niri msg -j event-stream 2>/dev/null | while read -r line; do
        echo "$line" | grep -q "WindowFocusChanged" || continue

        FOCUSED_ID=$(echo "$line" | grep -oP '"id":\s*\K\d+|null')

        if [ "$FOCUSED_ID" = "null" ] || [ -z "$FOCUSED_ID" ]; then
            APP_ID=""
        else
            APP_ID=$(niri msg -j focused-window 2>/dev/null | grep -oP '"app_id":\s*"\K[^"]*')
        fi

        if [ "$APP_ID" = "pcoip-client" ]; then
            if [ "$CURRENT" != "pcoip" ]; then
                cp "$NIRI_CONFIG" "$NORMAL"
                cp "$PCOIP" "$NIRI_CONFIG"
                CURRENT="pcoip"
                echo "ON" > "$STATE_FILE"
                echo "[pcoip] Switched to PCoIP config"
            fi
        else
            if [ "$CURRENT" != "normal" ]; then
                cp "$NORMAL" "$NIRI_CONFIG"
                CURRENT="normal"
                echo "OFF" > "$STATE_FILE"
                echo "[pcoip] Restored normal config"
            fi
        fi
    done
}

cleanup() {
    kill $WATCH_PID 2>/dev/null
    wait $WATCH_PID 2>/dev/null

    if [ "$(cat "$STATE_FILE" 2>/dev/null)" = "ON" ]; then
        cp "$NORMAL" "$NIRI_CONFIG"
        echo "[pcoip] Cleanup: restored normal config"
    fi
    rm -f "$STATE_FILE"
}

echo "OFF" > "$STATE_FILE"
focus_watch &
WATCH_PID=$!
trap cleanup EXIT INT TERM

# Unset KDE/Plasma Qt env vars that conflict with pcoip-client bundled Qt 6.9.3.
# The package wrapper (/usr/bin/pcoip-client) picks native Wayland automatically.
unset QT_STYLE_OVERRIDE
unset QT_SCALE_FACTOR_ROUNDING_POLICY
unset QT_AUTO_SCREEN_SCALE_FACTOR
unset QT_ENABLE_HIGHDPI_SCALING
unset QT_QPA_PLATFORMTHEME
unset QT_QPA_PLATFORMTHEME_QT6

exec pcoip-client "$@"
```

```bash
chmod +x ~/.local/bin/pcoip
```

---

## Usage

```bash
pcoip
```

| State | Mod binds | Alt+Tab | Mod+Mouse |
|-------|-----------|:-------:|:---------:|
| **PCoIP focused** | Whitelist only (~17 active) | → remote | → remote |
| **Other window** | All 110+ normal | Niri switcher | No-op (permanent) |

> **Note:** Niri's `toggle-keyboard-shortcuts-inhibit` escape hatch only toggles an inhibitor the app already requested; HP's client never requests `zwp_keyboard_shortcuts_inhibit_v1`, so the config-swap approach above is required. The client now runs natively on Wayland (no XWayland/xwayland-satellite in the input path).
