# KDE Plasma Theming (Optional)

> This section was extracted from the main installation guide.
> Only apply if using KDE Plasma — skip for Niri + Noctalia setups.

## Kvantum + Vinyl

```bash
sudo pacman -S --noconfirm --needed kvantum

# Vinyl theme (AUR)
yay -S --noconfirm --needed vinyl-git
```

## Theme

🎨 These are my personal favorites — use them as a starting point and change anything you don't like.

Stutter note: Whitesur, Orchis, and Colloid themes caused window-resize stutter on my system. Qogir, Darkly, and Vinyl didn't. Hardware may change results, so try a few before settling.

### WhiteSur KDE + GTK

WhiteSur is a macOS-inspired theme with light and dark variants. It needs both the KDE and GTK parts.

```bash
repo=/tmp/WhiteSur-kde
rm -rf $repo
git clone https://github.com/vinceliuice/WhiteSur-kde.git $repo
cd $repo && ./install.sh
```

```bash
repo=/tmp/WhiteSur-gtk
rm -rf $repo
git clone https://github.com/vinceliuice/WhiteSur-gtk-theme.git $repo
cd $repo && ./install.sh -c dark -c light
```

### Orchis KDE + GTK

Orchis is a Material Design-inspired theme. Same pattern — KDE first, then GTK.

```bash
# Orchis KDE
repo=/tmp/Orchis-kde
rm -rf $repo
git clone https://github.com/vinceliuice/Orchis-kde.git $repo
cd $repo && ./install.sh

# Orchis GTK
repo=/tmp/Orchis-theme
rm -rf $repo
git clone https://github.com/vinceliuice/Orchis-theme.git $repo
cd $repo && ./install.sh -c dark -c light
```

### Colloid KDE + Icons

Colloid is a modern, clean theme based on the Nordic color palette.

```bash
# Colloid KDE
repo=/tmp/Colloid-kde
rm -rf $repo
git clone https://github.com/vinceliuice/Colloid-kde.git $repo
cd $repo && ./install.sh

# Colloid icon theme
repo=/tmp/Colloid-icon-theme
rm -rf $repo
git clone https://github.com/vinceliuice/Colloid-icon-theme.git $repo
cd $repo && ./install.sh
```
