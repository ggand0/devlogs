# 015 macOS App Bundle Icon: eframe Runtime Override

Branch: `main` (direct commits)

## Summary

The macOS `.app` bundle built with `cargo bundle` showed the default egui icon instead of the ViewSkater icon, despite the `.icns` file being correctly embedded in the bundle. Root cause: eframe overrides the macOS bundle icon at runtime by calling `NSApplication.setApplicationIconImage()` with its built-in egui icon. The fix was passing the ViewSkater icon programmatically via `ViewportBuilder::with_icon()`.

## Problem

After setting up `cargo bundle` for the egui version (copying the `[package.metadata.bundle]` config and `ViewSkater.icns` from the iced version), the bundled `.app` displayed the default egui icon in the Dock and Finder — not the ViewSkater icon.

The bundle itself was correct:
- `Info.plist` had `CFBundleIconFile = ViewSkater.icns`
- `Contents/Resources/ViewSkater.icns` was present (311KB)

## Investigation

### eframe's runtime icon override

eframe's `epi_integration.rs` manages app icon state via `AppTitleIconSetter`:

1. On startup, checks `NativeOptions::viewport::icon` for a custom icon
2. If no custom icon is provided, calls `load_default_egui_icon()` which embeds `eframe/data/icon.png` (the egui logo)
3. Every frame in `pre_update()`, calls `set_title_and_icon_mac()` which:
   - Creates an `NSImage` from the icon's PNG bytes
   - Calls `NSApplication.setApplicationIconImage()` to set it

This means **the macOS `.icns` in the bundle is always overridden at runtime** by whatever eframe has — and without `with_icon()`, that's the egui default.

Key code path in `eframe/src/native/app_icon.rs`:
```rust
fn set_title_and_icon_mac(title: &str, icon_data: Option<&IconData>) -> AppIconStatus {
    // ...
    let data = NSData::from_vec(png_bytes);
    let app_icon = NSImage::initWithData(NSImage::alloc(), &data);
    app.setApplicationIconImage(app_icon.as_deref());  // overrides bundle icon
}
```

### Why the iced version didn't have this problem

The iced version uses winit directly and calls `window.set_window_icon()` with a loaded icon. It doesn't have an eframe-like layer that auto-sets an application icon every frame. The `.icns` in the bundle is the only icon source, so it works as expected.

## Fix

### 1. Added icon PNG to assets

Copied `icon_256.png` (the 256x256 ViewSkater icon) from the iced version's `assets/` into `viewskater-egui/assets/`.

### 2. Load and pass icon via `ViewportBuilder::with_icon()` (`src/main.rs`)

```rust
fn load_icon() -> Option<egui::IconData> {
    let image = image::load_from_memory(include_bytes!("../assets/icon_256.png"))
        .ok()?
        .into_rgba8();
    let (width, height) = image.dimensions();
    Some(egui::IconData {
        rgba: image.into_raw(),
        width,
        height,
    })
}

fn main() -> eframe::Result {
    // ...
    let mut viewport = egui::ViewportBuilder::default()
        .with_inner_size([1280.0, 720.0])
        .with_drag_and_drop(true);

    if let Some(icon) = load_icon() {
        viewport = viewport.with_icon(std::sync::Arc::new(icon));
    }

    let options = eframe::NativeOptions {
        viewport,
        ..Default::default()
    };
    // ...
}
```

The icon is embedded at compile time via `include_bytes!`. eframe's `AppTitleIconSetter` now uses this icon instead of the default egui one, so `NSApplication.setApplicationIconImage()` sets the correct ViewSkater icon.

### 3. Added `[package.metadata.bundle]` to `Cargo.toml`

```toml
[package.metadata.bundle]
name = "ViewSkater"
identifier = "com.ggando.viewskater-egui"
icon = ["assets/ViewSkater.icns"]
short_description = "A fast image viewer built with egui"
```

This configures `cargo bundle` to produce the `.app` with the correct name, identifier, and `.icns` file. The `.icns` is still needed for Finder icons and other places where macOS reads from the bundle rather than the runtime `NSImage`.

### Files changed

- `src/main.rs` — Added `load_icon()` and `ViewportBuilder::with_icon()`
- `assets/icon_256.png` — 256x256 ViewSkater icon (copied from iced version)
- `assets/ViewSkater.icns` — macOS icon set (copied from iced version)
- `Cargo.toml` — Added `[package.metadata.bundle]` section

## macOS icon resolution: bundle vs runtime

| Source | Used by | In this app |
|--------|---------|-------------|
| `Contents/Resources/ViewSkater.icns` (bundle) | Finder, Get Info, file associations | Set via `cargo bundle` + `Cargo.toml` metadata |
| `NSApplication.setApplicationIconImage()` (runtime) | Dock, app switcher, About panel | Set by eframe from `ViewportBuilder::with_icon()` |

Both are needed for a fully correct icon experience on macOS. The `.icns` covers static/Finder contexts; the runtime `NSImage` covers the Dock and app switcher while the app is running.

## Bundling command reference

```sh
cargo bundle --release
# Output: target/release/bundle/osx/ViewSkater.app

# Optional: create .dmg for distribution
cd target/release/bundle/osx
hdiutil create -volname "ViewSkater" -srcfolder "ViewSkater.app" -ov -format UDZO "ViewSkater.dmg"
```

## Relation to 014 (Linux app icon)

014 addressed icon visibility on Linux where the issue was GNOME's desktop-file-based icon resolution ignoring `_NET_WM_ICON`. This entry addresses macOS where the issue is the opposite: the bundle icon is correct but eframe overrides it at runtime. Both platforms now have working icons, through different mechanisms:

| Platform | Mechanism |
|----------|-----------|
| **macOS** | `ViewportBuilder::with_icon()` → eframe sets `NSApplication.setApplicationIconImage()` + `.icns` in bundle |
| **Linux** | `with_app_id()` + `.desktop` file with `StartupWMClass` + hicolor icon theme |
| **Windows** | `ViewportBuilder::with_icon()` → eframe/winit sets `WM_SETICON` |
