# UI Chrome: Menu Bar, Footer, About Modal, Settings Modal, and Window Size Fix

**Date:** 2026-03-13
**Branch:** `feat/ui-chrome` (based on `main`)

## Overview

This branch adds the application UI chrome to match the iced version: a menu bar with File/Edit/View/Help menus, a footer status bar, an About modal with compile-time build info, a Settings modal with persisted preferences, and a fix for window sizing on scaled displays.

## Commits

| Hash | Description |
|------|-------------|
| `796c96d` | Add menu bar, footer, and file dialogs |
| `255525c` | Add about modal with build info and clickable GitHub link |
| `ed8b3ef` | Fix default window size to 720p, hide cache overlay by default, fix overlay positions |
| `001735e` | Fix window size to use physical pixels like iced version |
| `3b5845c` | Add settings modal with display toggles and performance sliders |

## New files

### `src/menu.rs` (~175 lines)

All menu bar and footer rendering, plus the `MenuAction` enum and a custom toggle switch widget.

**Menu bar** (`show_menu_bar`):
- **File**: Open Folder (Ctrl+Shift+O), Open File (Ctrl+O) — both with pane-targeting submenus in dual mode — Close (Ctrl+W), Quit (Ctrl+Q)
- **Edit**: Preferences (opens settings modal)
- **View**: Single/Dual Pane radio buttons (Ctrl+1/2), toggle switches for Footer (Tab), FPS Overlay, Cache Overlay
- **Help**: About

Returns a `MenuAction` enum variant that `App::handle_menu_action` dispatches. This avoids mutable borrow conflicts — the menu bar only reads panes and UI state, then the returned action is applied separately.

**Footer** (`show_footer`):
- Bottom panel showing filename (monospace, light gray), resolution from current texture, and file size
- File size formatted as B/KB/MB via `format_file_size`

**Toggle switch** (`toggle_switch`):
- iOS/Steam-style animated toggle: blue background when on, gray when off
- Uses `animate_bool_with_time` for smooth 150ms knob transition
- 32x16px, drawn manually with `rect_filled` (track) and `circle_filled` (knob)

### `build.rs` (~42 lines)

Captures compile-time information via `cargo:rustc-env` directives:

| Variable | Source | Example |
|----------|--------|---------|
| `BUILD_TIMESTAMP` | `chrono::Utc::now()` | `20260313.143022` |
| `GIT_HASH` | `git rev-parse HEAD` | `001735e...` (full) |
| `GIT_HASH_SHORT` | First 7 chars of hash | `001735e` |
| `TARGET_PLATFORM` | `CARGO_CFG_TARGET_ARCH`-`CARGO_CFG_TARGET_OS` | `x86_64-linux` |
| `BUILD_PROFILE` | `PROFILE` env var | `release` |
| `BUILD_STRING` | version.timestamp | `0.1.0.20260313.143022` |

Reruns on `.git/HEAD` and `.git/refs/heads/` changes so the hash stays current.

### `src/build_info.rs` (~31 lines)

`BuildInfo` struct with static methods that read the `env!()` macros set by `build.rs`. Provides `display_version()` for formatted output (`"0.1.0 (20260313.143022)"`).

This mirrors the iced version's `build_info.rs` but is simpler — no macOS bundle version or feature flag listing since the egui version doesn't have those yet.

## About modal implementation

The about modal in `App::show_about_modal` uses two `egui::Area` layers:

1. **Backdrop** (`Order::Foreground`): Full-screen semi-transparent black overlay (`alpha=200`). Clicking it dismisses the modal.
2. **Modal content** (`Order::Tooltip`): Centered card with the actual about info. Must be a higher order than the backdrop to render on top.

The `Order::Tooltip > Order::Foreground` layering was discovered after the initial implementation placed both at `Order::Foreground`, which caused the backdrop to paint over the modal content. In egui, areas at the same order level are painted in creation order, but the backdrop's full-screen `rect_filled` covered the modal area.

**Modal content layout** (matching iced version):
- "ViewSkater" title (25pt, bold)
- Version line with build timestamp (15pt)
- Build string with profile (12pt, muted gray)
- Git commit short hash (12pt, muted gray)
- Platform target (12pt, muted gray)
- Author: "Gota Gando" (15pt, blue accent `rgb(100, 160, 240)`)
- Clickable GitHub link — opens via `webbrowser::open()`, blue text with pointer cursor

**Dismissal**: Click outside (backdrop) or press Escape.

## Settings modal and persistence

### `src/settings.rs` (~155 lines)

`AppSettings` struct with 5 fields, `serde`-driven YAML persistence, and the modal UI.

**Settings fields:**

| Setting | Type | Default | Range | Description |
|---------|------|---------|-------|-------------|
| `show_footer` | bool | `true` | — | Footer status bar visibility |
| `show_fps` | bool | `true` | — | FPS overlay visibility |
| `show_cache_overlay` | bool | `false` | — | Cache debug overlay visibility |
| `cache_count` | usize | `5` | 1–20 | Sliding window half-size (total window = 2n+1) |
| `lru_capacity` | usize | `50` | 10–200 | Decode LRU cache max entries |

The struct uses `#[serde(default)]` so new fields added in future versions get their defaults when loading an older config file — forward-compatible YAML serialization.

**Persistence:**
- Config path: `dirs::config_dir() / "viewskater-egui" / "settings.yaml"` (typically `~/.config/viewskater-egui/settings.yaml` on Linux)
- Loaded once on startup in `App::new`
- Saved when the settings modal is dismissed (Escape or click-outside) and when View menu toggles change
- `load()` falls back to `Default::default()` on missing file or parse error — first launch just works

**Modal UI layout:**

Uses the same two-layer `egui::Area` approach as the About modal (backdrop at `Order::Foreground`, card at `Order::Tooltip`). Single-page layout (~400px wide) with two sections:

1. **Display** — 3 toggle switches (reusing `toggle_switch` from `menu.rs`, now `pub(crate)`)
2. **Performance** — 2 `egui::Slider` widgets for cache sizes

Each section is wrapped in a darker `Frame` (`from_gray(30)`) inside the card (`from_gray(40)`) to create visual grouping.

### Dynamic cache resizing

The performance sliders apply immediately — each frame the slider value changes, `apply_settings_to_caches()` is called which:

1. `SlidingWindowCache::set_cache_count()` — sets the new half-size and calls `initialize()`, which clears all slots, synchronously decodes the center image, and spawns background loads for the new window
2. `DecodeLruCache::set_capacity()` — sets the new max and evicts LRU entries if the cache exceeds the new limit

No throttling is needed because `set_cache_count` triggers a synchronous decode that blocks the frame — the slider can only advance one step per decode, making it self-throttling.

The parameterization required changing the cache constructors from hardcoded constants to parameters:
- `SlidingWindowCache::new(ctx)` → `SlidingWindowCache::new(ctx, cache_count)`
- `DecodeLruCache::new()` → `DecodeLruCache::new(capacity)`

`Pane` now stores `cache_count` and `lru_capacity` so it can use them when creating new caches in `open_path`.

### Dependencies added for settings

| Crate | Type | Purpose |
|-------|------|---------|
| `serde` 1.x + `derive` | runtime | Serialize/deserialize `AppSettings` |
| `serde_yaml` 0.9 | runtime | YAML config file format |
| `dirs` 6.x | runtime | Platform config directory resolution |

## Window size: logical vs physical pixels

### The problem

The iced version sets window size with `winit::dpi::PhysicalSize::new(width, height)`, which specifies exact physical pixel dimensions regardless of display scaling. egui's `ViewportBuilder::with_inner_size` takes logical points (egui's internal coordinate system). On a display with fractional scaling:

```
logical_size × scale_factor = physical_size
1280 × 1.25 = 1600 (physical width)
 720 × 1.25 =  900 (physical height)
```

So `with_inner_size([1280.0, 720.0])` produced a 1600×900 physical pixel window on a 1.25× scaled display, not the intended 720p.

### The fix

egui's `ViewportBuilder` has no way to specify physical pixel sizes. The solution is a first-frame resize:

```rust
if !self.initial_size_set {
    if let Some(ppp) = ctx.input(|i| i.viewport().native_pixels_per_point) {
        if (ppp - 1.0).abs() > 0.01 {
            let logical = egui::vec2(
                DEFAULT_WINDOW_WIDTH / ppp,   // 1280 / 1.25 = 1024
                DEFAULT_WINDOW_HEIGHT / ppp,  //  720 / 1.25 = 576
            );
            ctx.send_viewport_cmd(egui::ViewportCommand::InnerSize(logical));
        }
    }
    self.initial_size_set = true;
}
```

The `ViewportBuilder` still specifies `[1280.0, 720.0]` as the initial size, which is correct for 1.0× displays. On scaled displays, the first frame reads `native_pixels_per_point` from winit and sends a resize command with the adjusted logical size. The target constants (`DEFAULT_WINDOW_WIDTH/HEIGHT`) are defined in physical pixels to match iced's behavior.

### Why not just use 1024×576 in ViewportBuilder?

That would produce the correct size on 1.25× displays but be wrong on 1.0× displays (too small), 1.5× displays (still wrong), and 2.0× Retina displays (tiny window). The first-frame resize adapts to any scale factor.

## Overlay position fix

Both the FPS overlay (`perf.rs`) and cache debug overlay (`cache.rs`) were anchored at `y=10.0` from the top of the screen rect. Since egui's `anchor` positions are relative to the full window (including the menu bar area), the overlays overlapped with the menu bar. Changed both to `y=36.0` to clear the ~24px menu bar.

Cache overlay also changed to hidden by default (`show_cache_overlay: false`) since it's a debug tool.

## Dependencies added

| Crate | Type | Purpose |
|-------|------|---------|
| `webbrowser` 1.x | runtime | Open GitHub link in browser from About modal |
| `chrono` 0.4 | build | Generate build timestamp in `build.rs` |
| `serde` 1.x + `derive` | runtime | Serialize/deserialize `AppSettings` |
| `serde_yaml` 0.9 | runtime | YAML config file format |
| `dirs` 6.x | runtime | Platform config directory resolution |

## File layout after this branch

```
main.rs         40 lines   CLI args, eframe bootstrap, mod declarations
app.rs         ~650 lines  App struct, keyboard/DnD/slider/panel UI, about modal, window sizing, settings wiring
settings.rs    ~155 lines  AppSettings struct, YAML persistence, settings modal UI
pane.rs        ~320 lines  Pane struct, navigation, loading, zoom/pan, rendering
menu.rs        ~185 lines  Menu bar (File/Edit/View/Help), footer, toggle switch, MenuAction enum
build_info.rs   31 lines   Compile-time build info accessors
build.rs        42 lines   Build script capturing git hash, timestamp, platform
decode.rs       44 lines   DynamicImage → ColorImage bypass
cache.rs       ~500 lines  SlidingWindowCache, SliderLoader, DecodeLruCache (parameterized)
file_io.rs      53 lines   Path resolution, image enumeration
perf.rs         85 lines   FPS overlay
```

## Remaining work on this branch

- Merge to `main`
