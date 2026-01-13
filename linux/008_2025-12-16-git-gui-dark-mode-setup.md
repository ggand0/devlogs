# git-gui Dark Mode Setup on Ubuntu

**Date:** 2025-12-16

## Problem

git-gui uses Tcl/Tk which doesn't respect Ubuntu's system-wide dark mode settings. The default theme is bright white and hard on the eyes.

## Solution

Install the **awthemes** Tk theme package and configure git-gui to use the `awdark` theme.

## Steps Performed

### 1. Download and Install awthemes

```bash
mkdir -p ~/.local/share/tk-themes
cd ~/.local/share/tk-themes
curl -L -o awthemes.zip "https://sourceforge.net/projects/tcl-awthemes/files/latest/download"
unzip -o awthemes.zip
```

Installed to: `~/.local/share/tk-themes/awthemes-10.4.0/`

### 2. Configure Environment Variable

Added to `~/.profile`:

```bash
export TCLLIBPATH="$HOME/.local/share/tk-themes/awthemes-10.4.0"
```

### 3. Configure X Resources

Added to `~/.Xresources`:

```
*TkTheme: awdark
```

Applied immediately with:

```bash
xrdb -merge ~/.Xresources
```

## Activation

- **Immediate:** Run `source ~/.profile` then launch `git gui`
- **Permanent:** Log out and back in, or reboot

## Available Themes

The awthemes package includes several themes:
- `awdark` - Dark theme (recommended)
- `awlight` - Light theme
- `awblack` - High contrast dark
- `awbreeze` - KDE Breeze style
- `awbreezedark` - KDE Breeze dark style
- `awarc` - Arc theme style
- `awclearlooks` - Clearlooks style

To switch themes, edit `~/.Xresources`:

```
*TkTheme: awbreezedark
```

Then run `xrdb -merge ~/.Xresources`.

## Notes

- This also applies to `gitk` and other Tk-based applications
- Menu bars and window title bars follow system theme (controlled by GTK/GNOME settings)
- If the theme doesn't apply, ensure TCLLIBPATH points to the correct directory containing `pkgIndex.tcl`

## References

- [Applying TK Themes to Git Gui](https://blog.serindu.com/2019/03/07/applying-tk-themes-to-git-gui/)
- [awthemes on SourceForge](https://sourceforge.net/projects/tcl-awthemes/)
