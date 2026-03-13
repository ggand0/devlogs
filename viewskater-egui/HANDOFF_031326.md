# Handoff: viewskater-egui State as of 2026-03-13

## Project

egui reimplementation of [viewskater](https://github.com/ggand0/viewskater) (iced version at `../data-viewer/`). Public repo at https://github.com/ggand0/viewskater-egui.

## Local directory layout

| Path | Description |
|------|-------------|
| `.` (this repo) | egui viewskater |
| `../data-viewer/` | iced viewskater (reference, uses local iced fork for instrumentation) |
| `../iced/` | Custom iced fork with per-step timing instrumentation |
| `../winit/` | Custom winit fork (branch `custom-dnd-0.30.13`) for DnD cursor position |
| `../egui-winit-custom/` | Patched egui-winit 0.31.1 for custom DnD events (NOT a git repo locally, pushed to `ggand0/egui-winit-custom` on GitHub) |
| `../data/demo/` | Test images: `small_images/`, `1080p_PNG_3MB/`, `4k_PNG_10MB/` |

## Codebase architecture

```
main.rs       36 lines   CLI args (clap), eframe bootstrap
app.rs        ~400 lines App struct, eframe::App, coordinates panes
pane.rs       ~310 lines Pane struct, navigation, loading, zoom/pan, rendering
menu.rs       ~150 lines Menu bar, footer, toggle switch widget, file dialogs
decode.rs     44 lines   DynamicImage -> ColorImage bypass conversion
cache.rs      473 lines  SlidingWindowCache, SliderLoader, DecodeLruCache
file_io.rs    53 lines   Path resolution, image enumeration
perf.rs       85 lines   Image FPS tracker and overlay
```

### Key design decisions

- **Pane** is a self-contained image viewer (file list + caches + rendering). Multiple Pane instances for dual mode.
- **App** coordinates cross-pane concerns (synced nav, DnD routing, slider).
- **Three cache layers**: SlidingWindowCache (background preload, 11 slots), SliderLoader (10ms throttle), DecodeLruCache (LRU, 50 capacity).
- **Decode bypass**: `image_to_color_image()` constructs `Vec<Color32>` directly from decoded pixel data, bypassing image crate v0.25 CICP pipeline (~60ms savings) and egui's `from_rgba_unmultiplied` double-pass (~13ms savings). See devlog 009.
- **Navigation**: Normal A/D or arrows = single step per key press. Shift held = skate mode (advance every frame).

### Custom forks

- **winit** (`ggand0/winit`, branch `custom-dnd-0.30.13`): DnD events with cursor position. macOS `msg_send` import fix already applied and pushed.
- **egui-winit** (`ggand0/egui-winit-custom`, main): Forwards custom DnD events into egui input.
- Both referenced as git URLs in `Cargo.toml` `[patch.crates-io]`.

## Current branches

### `main` (up to date)

Recent commits:
- `4b73cf8` Add Shift modifier for skate mode
- `362c09d` Extract modules from main.rs and fix clippy warnings
- `76aacbc` Update README
- `660e662` Merge PR #1 (slider performance)

PR #1 was reproduced on a fresh repo after `git filter-repo` stripped Claude Co-Authored-By footers. See devlog 010 for the exact steps.

### `feat/ui-chrome` (active, based on main)

- `796c96d` Add menu bar, footer, and file dialogs
- Menu bar: File (Open Folder/File with pane targeting submenu, Close, Quit), View (Single/Dual pane radios, toggle switches for Footer/FPS/Cache overlay), Help (About placeholder)
- Footer: filename, resolution, file size
- Custom toggle switch widget (iOS/Steam style, blue bg when on, animated knob)
- File dialogs via `rfd` crate
- Close (Ctrl+W) resets all panes to empty state
- **TODO on this branch**: About modal, settings UI

## Performance (achieved)

| Scenario | egui | iced |
|----------|------|------|
| 4K slider cold cache | ~6.5-7 FPS | ~6.6 FPS |
| 4K slider warm cache | ~18 FPS | ~6.6 FPS |
| 4K keyboard nav | ~40-45 FPS | ~11 FPS |
| 1080p slider | ~30 FPS | ~25 FPS |
| macOS 4K slider | ~11 FPS | N/A |
| macOS 4K keyboard | ~25-30 FPS | N/A |

## Planned work (priority order)

1. **feat/ui-chrome** (in progress): About modal, settings UI, then merge to main
2. **Minor fixes on main**: window default size (currently 1280x720 in code but shows 900p), remaining keyboard shortcuts from iced version
3. **feat/dual-pane-improvements**: Individual slider per pane in dual mode + zoom clipping fix (image goes outside pane when zoomed)
4. **Future features** (each their own branch): COCO dataset vis, JP2 format, window state restoration, crash logging, fullscreen mode, archive file support, about modal, macOS Finder double-click support

## Future code organization

When Pane grows too large, decompose into:
```
Pane
  ImageCollection   file list, current index, navigation
  Viewport          zoom, pan, rendering
  CacheManager      SlidingWindowCache + DecodeLruCache + SliderLoader
```

Not needed yet at ~310 lines.

## Build and run

```bash
RUST_LOG=viewskater=debug cargo run --profile opt-dev -- /path/to/images/
```

opt-dev profile: `inherits = "release"`, `opt-level = 3`, `lto = false`, `codegen-units = 16`.

## Devlogs

| Entry | Topic |
|-------|-------|
| 000 | Project overview and architecture |
| 001-008 | Phase 1 through slider atlas analysis |
| 009 | Slider performance profiling, CICP bypass, side-by-side egui vs iced comparison |
| 010 | Refactoring plan, repo history cleanup procedure, future component design |

## User preferences

- No Claude Co-Authored-By footers in commits
- No `git add -A`, add individual files
- Present tense verb commit titles ("Add...", "Fix...", "Remove...")
- No devlog markdowns in commits
- Don't propose lazy/defeatist options
- Evidence-based profiling, don't assume bottlenecks
- The user plans to rename this repo to `viewskater` and the iced version to `viewskater-iced` after migration is complete
