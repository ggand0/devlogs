# 054 Linux App Icon: Desktop Integration & GNOME Icon Resolution

Branch: `main`

## Summary

Fix for the missing taskbar icon on Linux (Ubuntu 24.04 / GNOME 46). The app showed the default gear icon in the GNOME dock for both `cargo run` and AppImage builds. Root cause: GNOME 46+ resolves icons strictly through `.desktop` files matched by `WM_CLASS`/`app_id`, and ViewSkater wasn't setting either. Same fix previously applied to the egui version (see `../viewskater-egui/devlogs/014_linux_app_icon.md`).

## Problem

On Ubuntu 24.04 (GNOME 46), ViewSkater showed the default gear icon in the taskbar/dock. This affected both `cargo run` and AppImage builds.

The existing `load_icon()` + `set_window_icon()` code correctly sets `_NET_WM_ICON` on X11, but GNOME 46 no longer reads this property as a fallback. On Wayland, `set_window_icon()` is a complete no-op in winit.

## Root cause

GNOME 46 (Ubuntu 24.04) changed icon resolution behavior. Previously (GNOME 42 / Ubuntu 22.04), if no matching `.desktop` file existed, GNOME would fall back to reading `_NET_WM_ICON`. GNOME 46+ now falls back to a generic gear icon instead.

GNOME 46+ resolves taskbar icons strictly through:
1. Read window's `WM_CLASS` (X11) or `app_id` (Wayland)
2. Find matching `.desktop` file where `StartupWMClass` matches
3. Use the `.desktop` file's `Icon=` field to look up the icon from the hicolor icon theme

ViewSkater was missing both: no `WM_CLASS`/`app_id` was explicitly set, and no `.desktop` file with `StartupWMClass` existed.

Sources:
- [Bug #2078657 — GNOME no longer displays X11 window icons](https://bugs.launchpad.net/bugs/2078657)
- [Fix Missing App Icon in Dock in Ubuntu 24.04 — UbuntuHandbook](https://ubuntuhandbook.org/index.php/2024/04/missing-icon-dock-ubuntu-2404/)

## Fix

### 1. Set WM_CLASS / app_id via winit (`src/main.rs`)

Added `with_name("viewskater", "viewskater")` to `WindowAttributes` in the existing Linux `cfg` block:

```rust
#[cfg(target_os = "linux")]
use winit::platform::x11::WindowAttributesExtX11;

// ...

#[cfg(target_os = "linux")]
{
    window_attrs = window_attrs
        .with_maximized(CONFIG.window_state == WindowState::Maximized)
        .with_position(config_position)
        .with_name("viewskater", "viewskater");
}
```

`with_name(general, instance)` sets the X11 `WM_CLASS` property. Both `WindowAttributesExtX11` and `WindowAttributesExtWayland` implement `with_name` with identical signatures and both write to the same internal `platform_specific.name` field, so importing either trait is sufficient for both backends. The X11 trait was chosen since the method name is more obviously associated with `WM_CLASS`.

Verified with `xprop`:
```
WM_CLASS(STRING) = "viewskater", "viewskater"
```

### 2. Desktop file for system installation (`assets/viewskater.desktop`)

```ini
[Desktop Entry]
Name=ViewSkater
Exec=/path/to/viewskater %f
Icon=viewskater
Type=Application
Categories=Graphics;Viewer;
StartupWMClass=viewskater
MimeType=image/png;image/jpeg;image/webp;image/tiff;image/bmp;
```

Key fields:
- `StartupWMClass=viewskater` — matches the `WM_CLASS` set in code
- `Icon=viewskater` — GNOME looks this up from hicolor icon theme at `icons/hicolor/256x256/apps/viewskater.png`
- `%f` in Exec — allows opening files via the desktop entry
- `Exec` **must be an absolute path** or a binary in `$PATH` — Gio silently rejects the entire `.desktop` file if the command can't be resolved. This is the shipped template; users substitute their actual binary path at install time.

### 3. Desktop file for AppImage builds (`cargo-appimage.desktop`)

```ini
[Desktop Entry]
Name=ViewSkater
Exec=viewskater
Icon=icon
Type=Application
Categories=Graphics;Viewer;
StartupWMClass=viewskater
```

cargo-appimage v2.4.0 (already installed) copies this custom file into the AppImage instead of auto-generating one. The `StartupWMClass=viewskater` is the critical addition — without it, GNOME can't match the window to the desktop entry.

Note: `Icon=icon` here references the icon inside the AppImage's internal structure, but GNOME cannot read files inside the mounted AppImage filesystem. The bundled desktop file is for AppImage integration tools (AppImageLauncher, appimaged) that extract and install it to XDG locations.

### Installation for icon to appear

The `.desktop` file and icon must be installed to standard XDG locations. The `Exec` field must be edited to the actual binary path before copying:

```bash
# Edit Exec= line in the desktop file to point to the actual binary, then:
cp assets/viewskater.desktop ~/.local/share/applications/
mkdir -p ~/.local/share/icons/hicolor/256x256/apps/
cp assets/icon_256.png ~/.local/share/icons/hicolor/256x256/apps/viewskater.png
gtk-update-icon-cache -f -t ~/.local/share/icons/hicolor/
update-desktop-database ~/.local/share/applications/
```

This works for both `cargo run` and AppImage because both now set `WM_CLASS=viewskater`, matching `StartupWMClass=viewskater` in the installed desktop file.

## Gotcha: Gio silently rejects unresolvable Exec

During debugging, `desktop-file-validate` passed but `Gio.DesktopAppInfo.new('viewskater.desktop')` returned NULL. The cause: `Exec=viewskater` (bare command name) failed because `viewskater` was not in `$PATH`. Gio validates the `Exec` field and silently drops the entire desktop entry if the command can't be resolved — no error, no warning, just invisible.

The egui version worked because its desktop file had `Exec=/home/.../viewskater-egui %f` with a full absolute path.

This is not documented in the freedesktop spec (which only says Exec should be a program name or absolute path). It's a Gio/GNOME implementation detail.

## How icon resolution works on each platform

| Platform | Mechanism | What we do |
|----------|-----------|------------|
| **Windows** | `_NET_WM_ICON` equivalent via winit | `load_icon()` + `set_window_icon()` — winit handles it |
| **macOS** | `NSApplication.setApplicationIconImage` | `load_icon()` + `set_window_icon()` — winit handles it |
| **Linux X11** | `_NET_WM_ICON` (ignored by GNOME 46+) → `.desktop` file + `WM_CLASS` match | `with_name()` + installed `.desktop` + hicolor icon |
| **Linux Wayland** | `app_id` → `.desktop` file → `Icon=` field (no programmatic icon API) | `with_name()` + installed `.desktop` + hicolor icon |

## Files changed

- `src/main.rs` — Added `with_name("viewskater", "viewskater")` and `WindowAttributesExtX11` import
- `assets/viewskater.desktop` — Desktop file template for system installation
- `cargo-appimage.desktop` — Custom desktop file for AppImage builds
