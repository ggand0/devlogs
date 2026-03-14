# Handoff: viewskater-egui State as of 2026-03-14

## Project

egui reimplementation of [viewskater](https://github.com/ggand0/viewskater) (iced version at `../data-viewer/`). Public repo at https://github.com/ggand0/viewskater-egui.

## What changed since last handoff (2026-03-13)

Two PRs merged to `main`:

### PR #2: `feat/ui-chrome` (devlog 011)

Added application UI chrome matching the iced version:

- **Menu bar**: File (Open Folder/File with pane targeting, Close, Quit), Edit (Preferences), View (pane layout radios, toggle switches for Footer/FPS/Cache overlay), Help (About)
- **Footer**: filename, resolution, file size in monospace
- **About modal**: build info, clickable GitHub link, backdrop dismiss
- **Settings modal**: 3 display toggles + 2 performance sliders (cache size, LRU capacity), persisted as YAML via `serde`/`serde_yaml`/`dirs`
- **Theme system**: `UiTheme` struct centralizing all custom colors, teal dark preset
- **Custom widgets**: toggle switch (iOS-style animated), accent-colored sliders (nav + settings)
- **Menu hover highlights**: full-width with submenu arrow alignment
- **Window size fix**: first-frame resize to achieve physical pixel target on scaled displays

### PR #3: `feat/dual-pane` (devlog 012)

Dual-pane bugfixes, features, responsive UI, and refactoring:

**Bugfixes:**
- Right-pane-only navigation broken (empty pane blocking `all_ready` + slider tracking wrong pane)
- Zoomed images bleeding across panes (fixed with `ui.painter_at()` clip rect)
- Tab keybind conflict (was bound to both toggle_dual and toggle_footer)
- Synced slider snap-back when panes have different image counts

**Features:**
- Independent dual-pane mode (Ctrl+3) with per-pane sliders
- Pane selection (clickable strips + bare 1/2 keys) controlling keyboard nav scope
- Split footer with per-pane metadata and divider
- Responsive UI: footer, menu bar, and empty pane message progressively hide as window narrows
- Synced zoom/pan across panes (toggle in settings, default on)
- Persistent zoom/pan across navigation (only resets on double-click, open, close, or menu reset)
- Reset Zoom/Pan menu item in View menu

**Refactoring:**
- Split `app.rs` into `app.rs` (struct, UI rendering, eframe::App) + `app/handlers.rs` (keyboard, DnD, menu actions, slider, settings handlers)
- Deduplicated slider result logic onto `Pane` (`apply_slider_target`/`apply_slider_release`)
- Removed dead code (`_id_salt`, always-zero decode display, per-frame `Pane::new(0,0)` allocation)
- Tightened `pub` → `pub(crate)` across `Pane`, `MenuAction`, `DualPaneMode`, `ImagePerfTracker`

## Codebase architecture

```
main.rs            41 lines   CLI args (clap), eframe bootstrap
app.rs            496 lines   App struct, new(), DualPaneMode, SliderResult, paint_nav_slider,
                               UI rendering (title, slider panel, central panel), eframe::App impl
app/handlers.rs   300 lines   handle_keyboard, handle_dropped_files, handle_menu_action,
                               apply_slider_result_all/one, apply_settings_to_caches,
                               set_single/dual_pane, open_folder/file_dialog, close_images
pane.rs           386 lines   Pane struct, navigation, loading, zoom/pan, rendering,
                               apply_slider_target/release, responsive empty pane message
menu.rs           446 lines   Menu bar (File/Edit/View/Help), footer (split in dual-pane,
                               responsive text), toggle switch, hover_row, MenuAction enum
settings.rs       271 lines   AppSettings, YAML persistence, settings modal, accent_slider
theme.rs           80 lines   UiTheme struct, teal_dark preset, apply_to_visuals
about.rs          124 lines   About modal with build info
build_info.rs      31 lines   Compile-time build info accessors
cache.rs          496 lines   SlidingWindowCache, SliderLoader, DecodeLruCache (parameterized)
perf.rs            56 lines   Image FPS tracker (fps_text)
decode.rs          44 lines   DynamicImage → ColorImage bypass
file_io.rs         53 lines   Path resolution, image enumeration
build.rs           42 lines   Build script (git hash, timestamp, platform)
                 2824 lines total
```

### Key design patterns

- **`Pane`** is self-contained (file list + caches + rendering). Multiple instances for dual mode. All fields `pub(crate)`, `load_sync` and `slider_loader` private.
- **`App`** coordinates cross-pane concerns. `impl App` split across `app.rs` (UI) and `app/handlers.rs` (logic). Handlers use `pub(super)` visibility.
- **Three cache layers**: SlidingWindowCache (background preload), SliderLoader (10ms throttle), DecodeLruCache (LRU, configurable capacity).
- **`DualPaneMode`**: `Synced` (shared slider, all panes navigate together) vs `Independent` (per-pane sliders, pane selection controls nav).
- **Pane selection**: `selected: bool` on `Pane`, scoped to independent mode. `is_active` closure pattern for filtered navigation.
- **Deferred action pattern**: `MenuAction` enum returned from menu, applied outside the UI closure. `SliderResult` collected during CentralPanel closure, applied after.
- **Responsive text**: measure-before-render via `ui.fonts()` to avoid borrow conflicts.
- **Zoom/pan sync**: `show_image()` returns bool, app propagates values between panes via `split_at_mut(1)`.

### Custom forks

- **winit** (`ggand0/winit`, branch `custom-dnd-0.30.13`): DnD events with cursor position
- **egui-winit** (`ggand0/egui-winit-custom`, main): Forwards custom DnD events into egui input
- Both referenced as git URLs in `Cargo.toml` `[patch.crates-io]`

## Local directory layout

| Path | Description |
|------|-------------|
| `.` (this repo) | egui viewskater |
| `../data-viewer/` | iced viewskater (reference) |
| `../iced/` | Custom iced fork with per-step timing instrumentation |
| `../winit/` | Custom winit fork for DnD cursor position |
| `../egui-winit-custom/` | Patched egui-winit 0.31.1 |
| `../data/demo/` | Test images: `small_images/`, `1080p_PNG_3MB/`, `4k_PNG_10MB/` |

## Performance (achieved)

| Scenario | egui | iced |
|----------|------|------|
| 4K slider cold cache | ~6.5-7 FPS | ~6.6 FPS |
| 4K slider warm cache | ~18 FPS | ~6.6 FPS |
| 4K keyboard nav | ~40-45 FPS | ~11 FPS |
| 1080p slider | ~30 FPS | ~25 FPS |
| macOS 4K slider | ~11 FPS | N/A |
| macOS 4K keyboard | ~25-30 FPS | N/A |

## Potential future work

- COCO dataset visualization
- JP2 format support
- Window state restoration
- Crash logging
- Fullscreen mode
- Archive file support
- macOS Finder double-click support
- Rename repo to `viewskater`, iced version to `viewskater-iced`

## Build and run

```bash
RUST_LOG=viewskater=debug cargo run --profile opt-dev -- /path/to/images/
```

opt-dev profile: `inherits = "release"`, `opt-level = 3`, `lto = false`, `codegen-units = 16`.

## User preferences

- No Claude Co-Authored-By footers in commits
- No `git add -A`, add individual files
- Present tense verb commit titles ("Add...", "Fix...", "Remove...")
- No devlog markdowns in commits
- Don't propose lazy/defeatist options
- Evidence-based profiling, don't assume bottlenecks
