**Niri config** (`~/.config/niri/config.kdl`):

Write this complete, validated config to your user's home. All blocks use multi-line format — niri 26.04 requires this:

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
    default-column-width {
        fixed 1080
    }
    default-window-height {
        fixed 920
    }
}

debug {
    honor-xdg-activation-with-invalid-serial
}

layer-rule {
    match namespace="^noctalia-wallpaper"
    place-within-backdrop true
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
overview {
    workspace-shadow {
        off
    }
}

window-rule {
    background-effect {
        blur true
        xray false
    }
}
layer-rule {
    match namespace="^noctalia-"
    background-effect {
        xray false
    }
}
blur {
    passes 2
    offset 3.0
    noise 0.03
    saturation 1.0
}

binds {
    // Noctalia IPC
    Mod+D       { spawn-sh "noctalia msg panel-toggle launcher"; }
    Mod+S       { spawn-sh "noctalia msg panel-toggle control-center"; }
    Mod+Comma   { spawn-sh "noctalia msg settings-toggle"; }
    Mod+Tab     { toggle-overview; }
    Mod+Q       { close-window; }
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

    // Screenshots
    Print { spawn "spectacle -f"; }
    Ctrl+Print { spawn "spectacle -a"; }
    Alt+Print { spawn "spectacle -w"; }
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

> ⚠️ **niri 26.04 KDL parser:** All blocks must use multi-line format. Inline blocks with semicolons (e.g. `touchpad { tap; natural-scroll }`) cause parse errors. Also, use a single `binds` block — duplicate `binds` blocks are silently ignored.