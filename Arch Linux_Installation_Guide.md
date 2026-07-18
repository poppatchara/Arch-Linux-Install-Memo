**Niri config** (`~/.config/niri/config.kdl`):

Write this validated config to your user's home. **All blocks use multi-line format** — niri 26.04 requires this. Binds use the XKB key names (`Equal` not `Equals`, `Page_Up` not `PageUp`):

```bash
mkdir -p /home/pop/.config/niri
tee /home/pop/.config/niri/config.kdl <<'NIRI_EOF'
input {
    keyboard {
        xkb {
            layout "us,th"
            options "grp:lalt_lshift_toggle"
        }
        numlock
    }
    touchpad {
        tap
        natural-scroll
    }
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
}

layer-rule {
    match namespace="^noctalia-wallpaper"
    place-within-backdrop true
}

layer-rule {
    match namespace="^noctalia-"
    background-effect {
        xray false
    }
}

layout {
    gaps 16
    background-color "transparent"
    center-focused-column "never"
    preset-column-widths {
        proportion 0.33333
        proportion 0.5
        proportion 0.66667
    }
}

blur {
    passes 2
    offset 3.0
    noise 0.03
    saturation 1.0
}

binds {
    // ── Window Management ──
    Mod+Q {
        close-window
    }
    Mod+F {
        maximize-column
    }
    Mod+Shift+F {
        fullscreen-window
    }
    Mod+V {
        toggle-window-floating
    }
    Mod+R {
        switch-preset-column-width
    }
    Mod+W {
        toggle-column-tabbed-display
    }
    Mod+Equal {
        set-column-width "+10%"
    }
    Mod+Minus {
        set-column-width "-10%"
    }
    Mod+Shift+Equal {
        set-window-height "+10%"
    }
    Mod+Shift+Minus {
        set-window-height "-10%"
    }
    Mod+BracketLeft {
        consume-window-into-column
    }
    Mod+BracketRight {
        expel-window-from-column
    }
    Mod+C {
        center-column
    }

    // ── Focus ──
    Mod+Left {
        focus-column-left
    }
    Mod+Right {
        focus-column-right
    }
    Mod+Up {
        focus-window-up
    }
    Mod+Down {
        focus-window-down
    }
    Mod+H {
        focus-column-left
    }
    Mod+L {
        focus-column-right
    }
    Mod+K {
        focus-window-up
    }
    Mod+J {
        focus-window-down
    }

    // ── Move ──
    Mod+Ctrl+Left {
        move-column-left
    }
    Mod+Ctrl+Right {
        move-column-right
    }
    Mod+Ctrl+Up {
        move-window-up
    }
    Mod+Ctrl+Down {
        move-window-down
    }

    // ── Workspaces ──
    Mod+1 {
        focus-workspace 1
    }
    Mod+2 {
        focus-workspace 2
    }
    Mod+3 {
        focus-workspace 3
    }
    Mod+4 {
        focus-workspace 4
    }
    Mod+5 {
        focus-workspace 5
    }
    Mod+6 {
        focus-workspace 6
    }
    Mod+7 {
        focus-workspace 7
    }
    Mod+8 {
        focus-workspace 8
    }
    Mod+9 {
        focus-workspace 9
    }
    Mod+Page_Up {
        focus-workspace-up
    }
    Mod+Page_Down {
        focus-workspace-down
    }
    Mod+Ctrl+1 {
        move-column-to-workspace 1
    }
    Mod+Ctrl+2 {
        move-column-to-workspace 2
    }
    Mod+Ctrl+3 {
        move-column-to-workspace 3
    }
    Mod+Ctrl+4 {
        move-column-to-workspace 4
    }
    Mod+Ctrl+5 {
        move-column-to-workspace 5
    }
    Mod+Ctrl+6 {
        move-column-to-workspace 6
    }
    Mod+Ctrl+7 {
        move-column-to-workspace 7
    }
    Mod+Ctrl+8 {
        move-column-to-workspace 8
    }
    Mod+Ctrl+9 {
        move-column-to-workspace 9
    }

    // ── Monitor ──
    Mod+Shift+Left {
        focus-monitor-left
    }
    Mod+Shift+Right {
        focus-monitor-right
    }
    Mod+Shift+Ctrl+Left {
        move-column-to-monitor-left
    }
    Mod+Shift+Ctrl+Right {
        move-column-to-monitor-right
    }

    // ── Overview ──
    Mod+Tab {
        toggle-overview
    }

    // ── Noctalia IPC ──
    Mod+D {
        spawn-sh "noctalia msg panel-toggle launcher"
    }
    Mod+S {
        spawn-sh "noctalia msg panel-toggle control-center"
    }
    Mod+Comma {
        spawn-sh "noctalia msg settings-toggle"
    }

    // ── Audio & Brightness ──
    XF86AudioRaiseVolume {
        spawn-sh "noctalia msg volume-up; canberra-gtk-play -i audio-volume-change -d 'volume feedback'"
    }
    XF86AudioLowerVolume {
        spawn-sh "noctalia msg volume-down; canberra-gtk-play -i audio-volume-change -d 'volume feedback'"
    }
    XF86AudioMute {
        spawn-sh "noctalia msg volume-mute; canberra-gtk-play -i audio-volume-change -d 'volume feedback'"
    }
    XF86MonBrightnessUp {
        spawn-sh "noctalia msg brightness-up"
    }
    XF86MonBrightnessDown {
        spawn-sh "noctalia msg brightness-down"
    }

    // ── Apps ──
    Mod+Return {
        spawn "ghostty"
    }
    Mod+E {
        spawn "dolphin"
    }
    Mod+B {
        spawn "firefox"
    }

    // ── Screenshots ──
    Print {
        spawn "spectacle -f"
    }
    Ctrl+Print {
        spawn "spectacle -a"
    }
    Alt+Print {
        spawn "spectacle -w"
    }
    Mod+Shift+S {
        spawn "spectacle -r"
    }

    // ── Power ──
    Mod+Shift+E {
        quit
    }
    Mod+Shift+P {
        power-off-monitors
    }
}

environment {
    ELECTRON_OZONE_PLATFORM_HINT "auto"
    QT_QPA_PLATFORM "wayland"
    QT_QPA_PLATFORMTHEME "kde"
    QT_WAYLAND_DISABLE_WINDOWDECORATION "1"
    XDG_CURRENT_DESKTOP "niri"
    XDG_SESSION_TYPE "wayland"
}

cursor {
    xcursor-theme "capitaine-cursors"
    xcursor-size 24
}

hotkey-overlay {
    skip-at-startup
}
NIRI_EOF

chown -R pop:pop /home/pop/.config/niri
```

> ⚠️ **niri 26.04 KDL rules:** All blocks must use multi-line format (`{` opens a new line). No semicolons in multi-line blocks. XKB key names: `Equal` not `Equals`, `Page_Up` not `PageUp`, `Page_Down` not `PageDown`. Custom `binds` block replaces ALL defaults — every keybind must be explicit.