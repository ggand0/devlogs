# Project Overview

**Repo:** [viewskater-egui](https://github.com/ggand0/viewskater-egui)

## What this is

An egui reimplementation of [viewskater](https://github.com/ggand0/viewskater), a fast image viewer originally built with iced. The egui version reproduces the core rendering and navigation features while serving as a framework evaluation and foundation for a LeRobot dataset curation tool.

## Why egui over iced

The iced version uses the Elm architecture (Model/Message/update/view) with retained-mode rendering. egui is immediate mode: UI code reads and mutates state in the same call, no message passing. This makes layout more direct (just specify a rect and draw) but loses the separation guarantees Elm provides.

Key differences found during the port:
- **Layout**: iced has a widget tree with layout negotiation. egui gives you a rect and you subdivide it yourself. The dual-pane draggable divider was straightforward to implement by computing two rects from a fraction and using `allocate_rect` with `Sense::click_and_drag` for the divider interaction.
- **Image rendering performance**: egui ended up faster. 40-45 FPS keyboard-navigating 4K images vs ~11 FPS with iced. Slider scrubbing: ~7 FPS cold / ~18 FPS warm cache vs ~6.6 FPS iced. See devlog 009 for profiling details.
- **Image pipeline**: iced reads file bytes asynchronously and uploads raw RGBA to the GPU atlas via `copy_buffer_to_texture`. egui requires building a `Vec<Color32>` for `ColorImage`, which means CPU-side per-pixel work. We bypass the default double-pass pipeline by constructing `ColorImage` directly from decoded pixel data (see `decode.rs`).
- **Drag-and-drop**: Required forking both winit (`ggand0/winit`, branch `custom-dnd-0.30.13`) and egui-winit (`ggand0/egui-winit-custom`) to get DnD events with cursor position for pane targeting.

## Architecture

```
main.rs       CLI args, eframe bootstrap
app.rs        App struct, eframe::App impl, coordinates panes
pane.rs       Pane struct, self-contained image collection viewer
decode.rs     DynamicImage -> ColorImage bypass conversion
cache.rs      SlidingWindowCache, SliderLoader, DecodeLruCache
file_io.rs    Path resolution, image enumeration
perf.rs       Image FPS tracker and overlay
```

### App

Owns a `Vec<Pane>` and coordinates cross-pane concerns: synced keyboard navigation, drag-and-drop routing (which pane gets the drop based on cursor position relative to divider), slider driving all panes, window title updates.

The `update()` loop runs each frame:
1. Poll cache results for all panes
2. Handle dropped files
3. Handle keyboard input
4. Update window title
5. Render slider panel
6. Render central panel (single pane or dual with divider)
7. Render overlays (FPS, cache debug)

### Pane

A self-contained image viewer instance. Owns the file list, current index, texture handle, zoom/pan state, and all three cache layers. Multiple Pane instances operate independently for dual-pane mode.

Methods fall into three categories:
- **State mutation**: `open_path`, `navigate`, `jump_to`, `load_sync`
- **Queries**: `can_navigate_forward/backward`, `is_next_cached`
- **Rendering**: `show_content`, `show_image` (zoom/pan interaction + `painter.image()`)

### Caching

Three layers, each addressing a different access pattern:

1. **SlidingWindowCache** (cache.rs) - Background preloading for keyboard navigation. A fixed-size window (`cache_count * 2 + 1` slots, default 11) centered on the current image. Background threads decode neighbors via `spawn_load`. The window slides on navigation: pop the trailing edge, push a new slot on the leading edge, spawn decode for it. On jump (slider release, Home/End), reinitializes the entire window around the new position.

2. **SliderLoader** (cache.rs) - Throttle gate for slider drag. 10ms minimum between sync decodes, matching iced's pattern where the framework naturally rate-limits by only decoding the latest handle per render frame. Without throttle, every slider pixel change would trigger a blocking 90ms+ decode.

3. **DecodeLruCache** (cache.rs) - CPU-side LRU cache of decoded `ColorImage`, capacity 50. Keyed by file index. On slider revisits, skips the ~90ms decode entirely (only GPU upload remains at ~2-5ms). Cleared on directory change to prevent stale entries.

### Decode bypass

The image crate v0.25 introduced CICP color space handling that routes RGB->RGBA conversion through an f32 fallback path (~60ms for 4K). egui's `ColorImage::from_rgba_unmultiplied` then iterates all pixels again to build `Vec<Color32>`. Our `image_to_color_image()` in `decode.rs` bypasses both by pattern-matching `DynamicImage` variants and constructing `Vec<Color32>` directly from raw bytes in a single pass. This collapses ~73ms of conversion into ~13ms.

### Custom forks

- **winit** (`ggand0/winit`, branch `custom-dnd-0.30.13`): Adds DnD event data (cursor position, file paths) to winit's event loop. Based on winit 0.30.13.
- **egui-winit** (`ggand0/egui-winit-custom`): Patched `egui-winit` 0.31.1 (the egui<->winit bridge crate from emilk/egui) to forward the custom DnD events from the winit fork into egui's input system.

## Future: component-based Pane

As features grow (selection mode, annotation overlays, metadata panels), `Pane` will accumulate too many responsibilities. The planned decomposition:

```
Pane
  ImageCollection   file list, current index, navigation, open_path
  Viewport          zoom, pan, show_image, input handling
  CacheManager      SlidingWindowCache + DecodeLruCache + SliderLoader, load_sync
```

Pane becomes a thin coordinator that delegates to these components. Each is independently testable and grows in its own file. Selection logic goes in ImageCollection, annotation rendering in Viewport, prefetch strategies in CacheManager. Not needed yet at 302 lines.

## Devlog index

| Entry | Topic |
|-------|-------|
| 001 | Phase 1: minimal image viewer |
| 002 | Image rendering FPS measurement |
| 003 | Sliding window cache implementation |
| 004 | Cache debug overlay |
| 005 | Gated navigation (wait for cache) |
| 006 | Dual pane mode |
| 007 | egui layout scalability analysis |
| 008 | Slider atlas analysis |
| 009 | Slider performance profiling and optimization |
| 010 | Refactoring plan and repo cleanup |
