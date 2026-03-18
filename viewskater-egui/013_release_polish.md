# 013 Release Polish: App Icon, Large Image Support, Fullscreen Mode

Branch: `main` (direct commits)

## Summary

Three features implemented directly on main to close out the 0.1.0 feature set: cross-platform app icon, automatic downscaling for images exceeding GPU texture limits, and fullscreen mode with cursor proximity UI reveal.

## Commits

| Commit | Description |
|--------|-------------|
| `0540ef4` | Add cross-platform app icon |
| `11cc842` | Support images larger than 8192px by automatic downscaling |
| `3e51c68` | Implement fullscreen mode with F11 toggle and Escape exit |
| `306f53e` | Add fullscreen cursor proximity reveal and FPS overlay |
| `138445c` | Fix fullscreen top zone to match iced version platform-specific values |

## Feature 1: Cross-platform app icon

Embedded the 256x256 PNG icon at compile time using `include_bytes!` and passed it to `ViewportBuilder::with_icon()`. This sets the window icon and taskbar icon on all platforms at the eframe level.

The iced version loads the icon via `winit::window::Icon::from_rgba()` and calls `window.set_window_icon()` directly on the winit window. In egui/eframe, `ViewportBuilder::with_icon()` abstracts this — it takes an `egui::IconData` (RGBA + dimensions) and eframe handles the winit conversion internally.

Note: `cargo run` on Linux (X11/Wayland) does not show the icon in the taskbar — this is expected behavior matching the iced version. The icon is visible after building a proper package (e.g. `.appimage` via `cargo-appimage`).

**Files changed:** `src/main.rs`, `assets/icon_256.png` (new), `assets/icon_48.png` (new), `assets/icon.ico` (new), `assets/ViewSkater.icns` (new)

## Feature 2: Support images >8192px

Added `downscale_if_needed()` in `decode.rs` that checks both dimensions against `MAX_TEXTURE_SIZE` (8192) before converting to `ColorImage`. Oversized images are proportionally downscaled using Lanczos3 filtering.

The fix is in `image_to_color_image()`, which all three decode paths flow through:
- `SlidingWindowCache::spawn_load()` — background thread preloading
- `SlidingWindowCache::decode_sync()` — synchronous initial load
- `Pane::load_sync()` — slider drag / jump fallback

This means no per-callsite changes were needed — a single insertion point covers everything.

The iced version also caps texture dimensions in `main.rs` at the wgpu surface configuration level (`size.width.min(MAX_TEXTURE_SIZE)`), which prevents a wgpu panic on surface resize. The egui version doesn't need this because eframe manages surface configuration internally.

**Files changed:** `src/decode.rs`

## Feature 3: Fullscreen mode

### Base implementation

Toggle fullscreen via F11 key or View > Fullscreen menu item. Escape exits fullscreen. Uses `ViewportCommand::Fullscreen(bool)` for borderless fullscreen. Menu bar, footer, and slider panel are hidden to provide an immersive view.

The iced version handles fullscreen at the raw winit level with platform-specific code paths:
- macOS: `WindowExtMacOS::set_simple_fullscreen()` instead of `window.set_fullscreen()` due to iced/winit compatibility
- Escape handling: intercepts `KeyEvent` before it reaches the iced state machine

In egui/eframe, `ViewportCommand::Fullscreen` handles all platform differences internally, and keyboard input is read uniformly via `ctx.input()`.

### Cursor proximity reveal

In fullscreen, the menu bar appears when the cursor approaches the top edge and the footer/slider appear near the bottom edge. This matches the iced version's UX.

**Zone sizes (matching iced):**
- Top zone: 200px on macOS/Windows, 50px on Linux (via `#[cfg]`)
- Bottom zone: 100px on all platforms

The macOS/Windows top zone is larger because fullscreen menu interactions (especially macOS's system menu bar area) need a more generous activation region.

### FPS overlay in fullscreen

When the FPS display setting is enabled, a semi-transparent floating badge renders in the top-right corner during fullscreen. Uses `ctx.layer_painter()` on a `Foreground` layer so it paints over the image content. In windowed mode, FPS continues to display in the menu bar as before.

**Files changed:** `src/app.rs`, `src/app/handlers.rs`, `src/menu.rs`

## Technical note: Logical vs physical pixels in cursor detection

The iced and egui versions handle cursor proximity detection in different coordinate spaces.

### iced version (physical pixels)

The iced version intercepts `WindowEvent::CursorMoved { position, .. }` from winit, which reports cursor position in **physical pixels**. The zone constants are therefore physical pixel values:

```rust
// main.rs — iced version
const FULLSCREEN_TOP_ZONE_HEIGHT: f64 = 50.0;    // physical pixels
const FULLSCREEN_BOTTOM_ZONE_HEIGHT: f64 = 100.0; // physical pixels

// In the winit event loop:
state.queue_message(Message::CursorOnTop(position.y < FULLSCREEN_TOP_ZONE_HEIGHT));
state.queue_message(Message::CursorOnFooter(
    position.y > (window.inner_size().height as f64 - FULLSCREEN_BOTTOM_ZONE_HEIGHT)));
```

### egui version (logical pixels)

The egui version uses `ctx.input(|i| i.pointer.hover_pos())` and `ctx.screen_rect()`, which both operate in **logical pixels** (points). The zone constants are therefore logical pixel values:

```rust
// app.rs — egui version
const FULLSCREEN_TOP_ZONE: f32 = 50.0;    // logical pixels
const FULLSCREEN_BOTTOM_ZONE: f32 = 100.0; // logical pixels

// In update():
let screen = ctx.screen_rect();
if let Some(pos) = i.pointer.hover_pos() {
    (pos.y - screen.min.y < FULLSCREEN_TOP_ZONE,
     screen.max.y - pos.y < FULLSCREEN_BOTTOM_ZONE)
}
```

### Practical difference

On a 1.25x scale display, the same numerical constant (50) means:
- **iced**: 50 physical pixels = 40 logical pixels
- **egui**: 50 logical pixels = 62.5 physical pixels

The zones end up slightly different in absolute screen distance, but for hover thresholds this is immaterial — the user experience is indistinguishable.

This logical-vs-physical difference is the same one that required the first-frame window size workaround in `app.rs`: egui's `with_inner_size` takes logical points, so on a 1.25x display, requesting `[1280, 720]` gives 1600x900 physical. The workaround divides by `native_pixels_per_point` to land at the intended physical size. The cursor proximity code doesn't need this workaround because both `hover_pos()` and `screen_rect()` are in the same (logical) coordinate space — the math is self-consistent.
