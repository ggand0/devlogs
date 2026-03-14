# UI Chrome: Menu Bar, Footer, Modals, Theme, Custom Sliders, and Polish

**Date:** 2026-03-13 – 2026-03-14
**Branch:** `feat/ui-chrome` (based on `main`)

## Overview

This branch adds the application UI chrome to match the iced version: a menu bar with File/Edit/View/Help menus, a footer status bar, an About modal with compile-time build info, a Settings modal with persisted preferences, and a fix for window sizing on scaled displays. Follow-up commits extract a centralized color theme, add full-width menu hover highlights with correct submenu arrow alignment, replace egui's built-in sliders with custom accent-colored sliders, and relocate the cache overlay to avoid menu overlap.

## Commits

| Hash | Description |
|------|-------------|
| `796c96d` | Add menu bar, footer, and file dialogs |
| `255525c` | Add about modal with build info and clickable GitHub link |
| `ed8b3ef` | Fix default window size to 720p, hide cache overlay by default, fix overlay positions |
| `001735e` | Fix window size to use physical pixels like iced version |
| `3b5845c` | Add settings modal with display toggles and performance sliders |
| `6316a77` | Extract color theme into UiTheme struct with teal dark preset |
| `b8a06b9` | Brighten menu text and add accent-tinted hover backgrounds |
| `03d764e` | Fix menu hover to light gray, add hover highlight to View menu items |
| `6cab332` | Make menu hover highlights fill full popup width including margins |
| `8efb089` | Replace built-in sliders with custom accent-colored sliders and fix accent color |
| `1ba3304` | Remove white border from slider handles and move cache overlay to right side |
| `6fdff63` | Align submenu arrow icons to right edge of menu popup |

## New files

### `src/theme.rs` (73 lines)

Centralized color theme for all custom UI elements. See [Centralized color theme](#centralized-color-theme-srcthemers) section below.

### `src/menu.rs` (~290 lines)

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

### `src/settings.rs` (~250 lines)

`AppSettings` struct with 5 fields, `serde`-driven YAML persistence, the modal UI, and a custom `accent_slider` widget.

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
2. **Performance** — 2 custom `accent_slider` widgets for cache sizes (see [Custom-drawn sliders](#custom-drawn-sliders))

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

## Centralized color theme (`src/theme.rs`)

All custom UI colors were scattered across `app.rs`, `menu.rs`, and `settings.rs`. The `UiTheme` struct centralizes them in one place:

```rust
pub struct UiTheme {
    pub accent: Color32,        // Primary accent (toggles, links, slider handles/fills)
    pub backdrop: Color32,      // Modal backdrop overlay
    pub card_bg: Color32,       // Modal/card background
    pub card_stroke: Color32,   // Modal/card border
    pub section_bg: Color32,    // Subsection background (darker than card)
    pub heading: Color32,       // Section heading text
    pub muted: Color32,         // Secondary/muted text
    pub toggle_off: Color32,    // Toggle switch off state
    pub toggle_knob: Color32,   // Toggle switch knob
    pub menu_hover: Color32,    // Menu item hover background
}
```

`UiTheme::teal_dark()` creates the preset matching the iced version. `apply_to_visuals()` wires the accent color into egui's built-in widget visuals (selection highlight, hyperlinks, active widget background, hover/open backgrounds).

### Accent color: iced parity

The iced version's visible accent is not the base primary `rgb(20, 148, 163)` but its `primary.strong` variant, which iced derives by lightening the base by 0.1 in HSL space. Replicating that transformation gives `rgb(26, 189, 208)` — the correct accent color.

## Menu hover highlights

egui's built-in menu hover backgrounds only cover the widget's allocated width, not the full popup. For a native-feeling menu, highlights need to span edge-to-edge including the popup's internal margins.

### Approach

Two helper functions in `menu.rs`:

1. **`setup_menu_hover(ui)`** — Called once per popup. Disables egui's built-in hover/active/open backgrounds by setting them to `TRANSPARENT`, then returns the popup's full rect coordinates (content area minus left margin offset, total width including both margins). The margin values come from `ui.style().spacing.menu_margin`.

2. **`hover_row(ui, id, theme, menu_left, menu_width, add_contents)`** — Wraps each menu item. Pre-allocates a `Shape::Noop` placeholder, renders the content via `ui.scope()`, then detects hover via raw pointer position (`pointer.hover_pos()`) over a full-width highlight rect. If hovered, the Noop is replaced with a filled rect at the menu_hover color. Using raw pointer position instead of egui's response system avoids stealing interaction from child widgets (submenu buttons, toggles).

### Submenu arrow alignment

`hover_row` initially used `ui.horizontal()` which constrained `menu_button` to minimum width, placing the submenu arrow "⏵" immediately after the text rather than at the right edge. Switching to `ui.scope()` lets `menu_button` inherit the menu's `top_down_justified` layout, filling available width and pushing the arrow to the right edge. Toggle switch rows (which need side-by-side layout for switch + label) get an explicit `ui.horizontal()` inside their closures.

## Custom-drawn sliders

egui's built-in `Slider` widget ties `widgets.inactive.bg_fill` to both the idle handle color and the rail background, making it impossible to have an accent-colored handle with a gray rail. All approaches to work around this (local style overrides, overpaint layers) had visual artifacts.

### Solution

Both slider locations (image navigation in `app.rs`, settings modal in `settings.rs`) use fully custom-drawn sliders:

1. **Allocate** a rect with `allocate_exact_size(size, Sense::drag())`
2. **Handle dragging** via `response.interact_pointer_pos()` — map pointer x to the value range, clamped to `[0, 1]` across the usable rail (inset by handle radius)
3. **Paint back-to-front**: unfilled rail (gray) → filled rail from left edge to handle position (accent) → handle circle (accent, no border)

The settings modal's `accent_slider` also renders the current value as monospace text to the right of the rail via `ui.put()`.

Key detail: the paint order matters. Painting the unfilled rail first, then the filled portion on top, then the handle circle on top of both, ensures clean visuals with no overlap artifacts.

## Cache overlay position

The cache debug overlay was anchored at `LEFT_TOP`, overlapping with File/View menu pulldowns. Moved to `RIGHT_TOP` with a y-offset of 66.0 (below the FPS overlay at 36.0) so both overlays stack on the right side without conflicting with menus or each other.

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
app.rs         ~690 lines  App struct, keyboard/DnD/custom slider/panel UI, about modal, window sizing, settings wiring
settings.rs    ~250 lines  AppSettings struct, YAML persistence, settings modal UI, accent_slider widget
theme.rs        73 lines   UiTheme struct, teal_dark preset, apply_to_visuals
pane.rs        ~320 lines  Pane struct, navigation, loading, zoom/pan, rendering
menu.rs        ~290 lines  Menu bar (File/Edit/View/Help), footer, toggle switch, hover_row, MenuAction enum
build_info.rs   31 lines   Compile-time build info accessors
build.rs        42 lines   Build script capturing git hash, timestamp, platform
decode.rs       44 lines   DynamicImage → ColorImage bypass
cache.rs       ~500 lines  SlidingWindowCache, SliderLoader, DecodeLruCache (parameterized)
file_io.rs      53 lines   Path resolution, image enumeration
perf.rs         85 lines   FPS overlay
```

## egui immediate-mode UI model

### Layout, interaction, and drawing in one pass

egui is immediate mode: there is no separate event/draw split like iced's `update()` → `view()`. The single `update()` function reads input, builds widgets, and records draw commands all in one frame. Each widget call like `ui.button("OK")` simultaneously:

1. **Allocates space** at the current layout cursor position
2. **Checks interaction** (clicked, hovered, dragged) against last frame's input
3. **Records shapes** to a draw list that egui renders at the end of the frame

### Cursor-based layout

The layout system works like a text cursor. `allocate_exact_size(size, sense)` reserves a rect at the current cursor position and advances the cursor. In a vertical layout, the cursor moves down; in `ui.horizontal(...)`, it moves right. You can't position widgets at arbitrary coordinates within a normal layout.

Escape hatches for free positioning:
- `egui::Area` — free-floating, positioned anywhere (modals, overlays)
- `ui.put(rect, widget)` — place a widget at a specific rect
- `ui.painter()` — draw shapes at any coordinate (no layout participation)
- `ui.allocate_new_ui(builder.max_rect(rect), ...)` — create a sub-UI region at a specific rect (dual-pane split)

### Custom widgets

Custom widgets follow the same allocate → interact → paint pattern as built-in ones. Example: the toggle switch in `menu.rs`:

```rust
let (rect, response) = ui.allocate_exact_size(size, Sense::click());  // 1. reserve space + register for clicks
if response.clicked() { *on = !*on; }                                  // 2. handle interaction
ui.painter().rect_filled(rect, radius, bg);                            // 3. draw track
ui.painter().circle_filled(knob_center, radius - 2.0, knob_color);    // 4. draw knob
```

`ui.painter()` returns a `Painter` — a low-level canvas API that appends shapes (rects, circles, lines, text) to a draw list. Built-in widgets like `ui.button()` use it internally too.

### Borrow-driven architecture patterns

egui closures (`ui.horizontal(|ui| {...})`, `panel.show(ctx, |ui| {...})`) borrow `&mut Ui`, which means you can't also mutate `self` inside them. Two patterns emerge:

1. **Self-contained** — interaction and drawing in one place. Works when the closure only needs to mutate data passed into it (e.g. `&mut settings`, `&mut bool`).

2. **Deferred action** — collect an enum value inside the closure, apply it after. `show_menu_bar` returns a `MenuAction` enum because the menu borrows `&panes` and `&mut settings`, so it can't also mutate panes to open files. `handle_menu_action` applies the action outside the borrow.

## Remaining work on this branch

- Merge to `main`
