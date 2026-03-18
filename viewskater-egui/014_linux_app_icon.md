# 014 Linux App Icon: Desktop Integration & GNOME Icon Resolution

Branch: `main` (direct commits)

## Summary

Investigation and fix for the missing taskbar icon on Linux (Ubuntu 24.04 / GNOME 46). The icon was set via `ViewportBuilder::with_icon()` but never appeared in the GNOME taskbar for both `cargo run` and AppImage builds. Root cause: on Linux, GNOME resolves icons through `.desktop` files matched by `WM_CLASS`/`app_id`, not through the X11 `_NET_WM_ICON` property. Additionally, the `WM_CLASS` was incorrect when running from an AppImage.

## Problem

After implementing the app icon in 013, the icon appeared in the window title bar on some platforms but never in the GNOME taskbar on Ubuntu 24.04. Both `cargo run` and the AppImage showed the default GNOME app icon. The iced version had the same problem, indicating this was not egui-specific.

## Investigation

### _NET_WM_ICON is not being set by winit

Despite `ViewportBuilder::with_icon()` passing valid icon data through the entire chain:
1. `load_icon()` → `egui::IconData` (256x256 RGBA) ✅
2. `to_winit_icon()` → `winit::window::Icon::from_rgba()` ✅
3. `WindowAttributes { window_icon: Some(RgbaIcon{...}) }` ✅
4. `UnownedWindow::new()` → `set_icon_inner()` → `xconn.change_property(_NET_WM_ICON, ...)` ✅ (code path executed)

The `_NET_WM_ICON` X11 property on the window was always empty (verified via `xprop`). Manual testing confirmed the property *can* be set (a test 2x2 icon set via xprop persisted), meaning the egui event loop doesn't clear it — winit's xcb `change_property` call silently fails for the 256x256 icon data (262KB). The exact failure point in x11rb was not identified; this appears to be a known limitation tracked in [egui#3992](https://github.com/emilk/egui/issues/3992).

Other apps on the same system (GTK file manager, etc.) successfully set `_NET_WM_ICON`, so the mechanism works in general — just not through the winit/x11rb path.

### GNOME 46 behavior change (why it worked on Ubuntu 22.04)

On Ubuntu 22.04 (GNOME 42), `_NET_WM_ICON` worked as a fallback — if there was no matching `.desktop` file, GNOME would still read the icon from the X11 property and display it in the taskbar (or show a duplicate dock entry with the icon).

**GNOME 46 (Ubuntu 24.04) changed this behavior.** If the window's `WM_CLASS` doesn't match any `.desktop` file's `StartupWMClass`, GNOME now falls back to a generic gear icon instead of reading `_NET_WM_ICON`. This is a deliberate behavior change, not a bug — GNOME is moving toward strict desktop-file-based icon resolution.

This explains why both the iced and egui versions of ViewSkater lost their taskbar icons after upgrading from 22.04 to 24.04 — the `_NET_WM_ICON` fallback path no longer exists.

Sources:
- [Bug #2078657 — GNOME no longer displays X11 window icons](https://bugs.launchpad.net/bugs/2078657)
- [Fix Missing App Icon in Dock in Ubuntu 24.04 — UbuntuHandbook](https://ubuntuhandbook.org/index.php/2024/04/missing-icon-dock-ubuntu-2404/)
- [egui#3992 — App icon not working on GNOME/Wayland](https://github.com/emilk/egui/issues/3992)

### GNOME icon resolution on Linux (current)

GNOME 46+ resolves taskbar icons strictly through:

1. Read window's `WM_CLASS` (X11) or `app_id` (Wayland)
2. Find matching `.desktop` file where `StartupWMClass` matches
3. Use the `.desktop` file's `Icon=` field to look up the icon from the hicolor icon theme

On Wayland, `set_window_icon()` is a complete no-op in winit — icon resolution is exclusively through the desktop file.

### WM_CLASS mismatch with AppImage

When running via AppImage, the `WM_CLASS` was set to `"viewskater-egui.AppImage"` (derived from the executable filename) instead of `"viewskater-egui"`. This broke the match with `StartupWMClass=viewskater-egui` in the desktop file.

### cargo-appimage v2.2.0 ignores custom desktop files

The `cargo-appimage` v2.2.0 binary always auto-generates a minimal `.desktop` file, ignoring any `./cargo-appimage.desktop` at the project root. The feature to check for and copy an existing custom desktop file was added in v2.4.0. Confirmed by comparing binary strings (v2.2.0 lacks the "Cannot find" error message from the copy-existing-file code path that v2.4.0 has).

## Fix

### 1. Set explicit app_id (`src/main.rs`)

Added `with_app_id("viewskater-egui")` to `ViewportBuilder`:

```rust
let mut viewport = egui::ViewportBuilder::default()
    .with_inner_size([1280.0, 720.0])
    .with_drag_and_drop(true)
    .with_app_id("viewskater-egui");
```

This normalizes `WM_CLASS` to `("", "viewskater-egui")` regardless of how the binary is invoked (directly, via AppImage, etc.). On Wayland, this sets the XDG `app_id` which compositors use for icon resolution.

### 2. Desktop file and icon installation

For GNOME to resolve the icon, a `.desktop` file and icon must exist in standard XDG locations:

- `~/.local/share/applications/viewskater-egui.desktop`
- `~/.local/share/icons/hicolor/256x256/apps/viewskater-egui.png`

The desktop file:
```ini
[Desktop Entry]
Name=ViewSkater
Exec=viewskater-egui
Icon=viewskater-egui
Type=Application
Categories=Graphics;Viewer;
StartupWMClass=viewskater-egui
```

Key: `Icon=viewskater-egui` (without path/extension) — GNOME looks this up from the hicolor icon theme at `icons/hicolor/256x256/apps/viewskater-egui.png`.

### 3. Upgrade cargo-appimage to v2.4.0

Upgraded from v2.2.0 to v2.4.0 so that the custom `cargo-appimage.desktop` at the project root is properly included in the AppImage. The v2.4.0 source confirms:

```rust
let mut desktop_entry_path = "./cargo-appimage.desktop";
// ...
if !std::path::Path::new(desktop_entry_path).exists() {
    // auto-generate
} else {
    std::fs::copy(desktop_entry_path, appdirpath.join("cargo-appimage.desktop"))?;
}
```

### Files changed

- `src/main.rs` — Added `with_app_id("viewskater-egui")`
- `cargo-appimage.desktop` — Custom desktop file for AppImage builds (includes `StartupWMClass`)
- `assets/viewskater-egui.desktop` — Desktop file template for system installation

## How icon resolution works on each platform

| Platform | Mechanism | What we do |
|----------|-----------|------------|
| **Windows** | `_WIN32 WM_SETICON` via eframe `app_icon.rs` | `ViewportBuilder::with_icon()` — eframe handles it |
| **macOS** | `NSApplication.setApplicationIconImage` via eframe | `ViewportBuilder::with_icon()` — eframe handles it |
| **Linux X11** | `_NET_WM_ICON` property (broken in winit/x11rb) → falls back to `.desktop` file + `WM_CLASS` match | `with_app_id()` + installed `.desktop` file + hicolor icon |
| **Linux Wayland** | `app_id` → `.desktop` file → `Icon=` field (no programmatic icon API) | `with_app_id()` + installed `.desktop` file + hicolor icon |

## Key takeaway

On Linux (both X11 and Wayland under GNOME), the only reliable way to show a taskbar icon is:
1. Set `app_id`/`WM_CLASS` in the application code
2. Install a `.desktop` file with matching `StartupWMClass` and `Icon=` reference
3. Install the icon to the hicolor icon theme

The `ViewportBuilder::with_icon()` approach works for Windows and macOS but is unreliable on Linux. This is a known limitation of the egui/winit ecosystem.

## AppImage distribution note

The AppImage bundles the `.desktop` file (via `cargo-appimage.desktop` at the project root) and icon internally, but GNOME cannot see files inside a mounted AppImage. The `.desktop` file and icon must be extracted and installed to XDG locations for the taskbar icon to appear.

The AppImage ecosystem handles this via external tools:
- **[AppImageLauncher](https://github.com/TheAssassin/AppImageLauncher)** — prompts to integrate on first run, auto-extracts `.desktop` and icon
- **appimaged** — daemon that watches directories and auto-integrates AppImages
- **Manual** — user installs the files themselves

For v0.1.0, manual installation steps are documented in the README. The app ships `assets/viewskater-egui.desktop` as a template.

Source: [AppImage Desktop Integration docs](https://docs.appimage.org/reference/desktop-integration.html)
