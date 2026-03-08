# Handoff: Image Atlas Cache for Slider Navigation

**Date:** 2026-03-08
**Goal:** Improve slider drag performance from ~4-5 FPS to match or exceed iced's ~7-8 FPS with 4K images.

## Prior context

Read these devlogs for background on the codebase and architectural decisions so far:

| Devlog | Relevant context |
|--------|-----------------|
| `devlogs/003_sliding_window_cache.md` | How the background decode + `mpsc` poll cache works |
| `devlogs/005_gated_navigation.md` | Gated navigation, why keyboard nav is fast (~25 FPS), comparison with iced's loading queue |
| `devlogs/006_dual_pane_mode.md` | Dual pane architecture, per-pane caches |
| `devlogs/007_egui_layout_scalability.md` | egui vs iced tradeoffs for rendering-heavy apps |
| `docs/plans/000_viewskater-egui-spec.md` | Original spec and motivation |
| `docs/plans/001_egui_vs_iced_for_lerobot_tool.md` | egui vs iced evaluation criteria |

## Local directory layout

| Path | Description |
|------|-------------|
| `.` (this repo) | egui viewskater — the project being improved |
| `../data-viewer/` | iced viewskater (production version) — reference for navigation logic, loading queues, synced split |
| `../iced/` | Custom iced fork — contains the atlas implementation at `wgpu/src/image/atlas.rs` |
| `../winit/` | Custom winit fork with DnD position fix (branch `custom-dnd-0.30.13`) |
| `../egui-winit-custom/` | Patched egui-winit for custom winit DnD events |
| `../data/demo/` | Test image directories: `small_images/`, `1080p_PNG_3MB/`, `4k_PNG_10MB/` |

## Problem

During slider drag, each position change triggers `load_sync()` — a full synchronous decode + GPU upload on the UI thread. For 4K images (~10MB PNG, ~90ms decode), this limits slider scrubbing to ~4-5 FPS. Keyboard navigation doesn't have this problem because the sliding window cache pre-decodes neighbors in background threads.

The iced version achieves ~7-8 FPS on the same images because iced's wgpu backend maintains an image atlas — a persistent GPU texture array where images are uploaded incrementally and retained across frames.

## Current architecture

### Slider path (the bottleneck)

`src/main.rs` — `show_bottom_panel()`:

```
slider.changed() → pane.load_sync(ctx) for each pane
slider.drag_stopped() → cache.jump_to() to rebuild window around final position
```

`load_sync()` does:
1. `image::open(path)` — decode file to `DynamicImage` (CPU, ~90ms for 4K PNG)
2. `img.to_rgba8()` — convert to RGBA (CPU, ~5ms)
3. `ctx.load_texture(name, color_image, LINEAR)` — upload to GPU via egui

Step 1 dominates. Step 3 creates a **new GPU texture** every call — the previous texture is dropped, and the new `ColorImage` (full RGBA8 bitmap) is uploaded from scratch.

### Keyboard nav path (fast, for reference)

Background threads decode images into `ColorImage`. `poll()` uploads completed images via `ctx.load_texture()`. Navigation only advances when the next image is already cached (`is_next_cached()` gate). Result: ~25 FPS for 4K keyboard nav.

### Key files

| File | Role |
|------|------|
| `src/main.rs` | `PaneState::load_sync()` (slider decode), `show_bottom_panel()` (slider UI) |
| `src/cache.rs` | `SlidingWindowCache` — 11-slot `VecDeque<Option<TextureHandle>>`, background thread pool via `mpsc`, `poll()` uploads |
| `src/perf.rs` | FPS overlay measuring image rendering throughput |

### egui texture pipeline

`ctx.load_texture(name, ColorImage, options)` → `TextureHandle`:
- Allocates a **new GPU texture per image** (separate `wgpu::Texture` or `glow::Texture`)
- Full RGBA8 upload every time, even if the same image size
- Previous texture is deallocated when `TextureHandle` is dropped
- No texture reuse, no atlas packing, no incremental updates

This is the fundamental inefficiency for slider scrubbing: every frame requires a full GPU texture allocation + upload cycle.

## How iced's atlas works (reference implementation)

Source: `/home/gota/ggando/rust_gui/iced/wgpu/src/image/atlas.rs` and siblings.

### Architecture

```
atlas.rs        — Atlas struct: wgpu::Texture (2D array), Vec<Layer>, CompressionStrategy
allocator.rs    — Guillotiere-based 2D bin packing within 2048x2048 layers
layer.rs        — Layer state: Empty / Busy(Allocator) / Full
entry.rs        — Entry: Contiguous(Allocation) or Fragmented { fragments, size }
cache.rs        — Cache around atlas, maps image handles to atlas entries
raster.rs       — Host-side decode + upload tracking
```

### Key mechanism

1. **Persistent GPU texture array**: one `wgpu::Texture` with `depth_or_array_layers` containing N layers of 2048x2048
2. **Bin packing**: images < 2048x2048 are packed into layers using guillotiere allocator. Images > 2048x2048 are fragmented across multiple layers.
3. **Incremental upload**: `copy_buffer_to_texture` writes only the new image's rect into the appropriate layer position — no full texture replacement
4. **Atlas growth**: when all layers are full, a new texture array is created with +1 layer and all existing layers are copied. This is the one stall point.
5. **Instanced rendering**: shader indexes into the texture array via layer index + normalized UV offset. Switching displayed image = updating instance data (sub-millisecond), not re-uploading textures.
6. **Optional BC1 compression**: `texpresso` crate compresses before upload (4x memory reduction, ~1-2ms per image)

### Why it's fast for slider scrubbing

When scrubbing through previously-viewed images, the atlas already contains them — display is just an instance buffer update with zero texture operations. For new images, only the new image's region is uploaded incrementally, not a full texture.

## Design space for egui

### Option A: Texture pool (simplest)

Instead of allocating a new GPU texture per `load_sync()`, maintain a pool of pre-allocated textures at a fixed size (e.g., 4K resolution). On slider drag, decode into a CPU buffer and upload into a reused texture via `ctx.load_texture()` with the same name, or use egui's `TextureHandle::set()` to replace contents in-place.

**Investigate**: does `TextureHandle::set(color_image, options)` reuse the same GPU texture or allocate a new one? If it reuses, this alone could cut GPU allocation overhead.

**Pros**: minimal code change, no custom GPU code
**Cons**: still decodes every image from disk on every slider step

### Option B: Decode cache (LRU) + texture pool

Add an LRU cache of decoded `ColorImage` buffers (CPU memory). On slider drag, check the LRU first — if the image was recently viewed, skip decode and only re-upload. Combined with texture reuse from Option A.

**Trade-off**: CPU memory. A 4K RGBA8 image is ~32MB. Caching 20 images = ~640MB. Could use downscaled previews for slider scrubbing (e.g., 1080p preview = ~8MB each).

### Option C: Background decode ahead of slider (predictive)

When slider drag starts, spawn background threads to decode images ahead of the drag direction. Similar to the keyboard nav cache but adapted for slider motion:
- Track slider velocity to predict which images will be needed
- Decode into a ring buffer of `ColorImage`
- On each slider step, check if the target image is already decoded

**Challenge**: slider drag is unpredictable — user can reverse direction, jump, or stop anywhere.

### Option D: Atlas (full iced approach)

Implement a texture atlas on the egui/glow side:
- Allocate a large 2D array texture via `glow` directly
- Pack images into layers using guillotiere
- On slider step, upload only the new image into an atlas slot
- Render by setting UV coordinates into the atlas

**Pros**: eliminates GPU allocation churn, enables instanced rendering
**Cons**: significant implementation effort, requires bypassing egui's texture management, directly using glow/wgpu APIs

### Option E: Thumbnail strip (orthogonal optimization)

Pre-generate small thumbnails (256px wide) for the entire directory on startup. Slider drag shows thumbnails (instant), slider release loads full-resolution image. This is a UX approach rather than a rendering optimization — the slider feels fast because it's showing cheap previews.

**Complements other options**: can combine with any of the above.

## Recommended investigation order

1. **Check `TextureHandle::set()` behavior** — does it reuse the GPU texture? If so, Option A is nearly free.
2. **Profile the decode vs upload split** — add timing to `load_sync()` to measure decode time vs `ctx.load_texture()` time separately. If upload is small (likely), the bottleneck is purely CPU decode.
3. **If decode-bound**: Option B (LRU of decoded images) or Option C (predictive decode) will help most.
4. **If upload-bound**: Option D (atlas) is needed, but this is unlikely given that upload of a single 4K RGBA8 texture should be <5ms.
5. **Option E** is worth prototyping regardless — it's a UX win even if rendering is fast.

## Benchmarking

Current FPS measurement is in `src/perf.rs` — `ImagePerfTracker` uses a 2-second rolling window of image display timestamps. For this work, you'll want to add per-step timing to `load_sync()` to separate:

- File I/O time (disk read)
- Decode time (`image::open`)
- RGBA conversion time (`to_rgba8`)
- GPU upload time (`ctx.load_texture`)

The sliding window cache's `spawn_load()` in `src/cache.rs` already shows the background decode pattern.

## Performance targets

| Scenario | Current egui | iced reference | Target |
|----------|-------------|----------------|--------|
| 4K PNG slider drag | ~4-5 FPS | ~7-8 FPS | >= 8 FPS |
| 1080p slider drag | ~15 FPS | ~20 FPS | >= 20 FPS |
| 4K keyboard nav | ~25 FPS | ~11.5 FPS | maintain |

## Key source paths

| Path | Description |
|------|-------------|
| `src/main.rs` — `PaneState::load_sync()` | Current slider decode path (sync decode + upload) |
| `src/main.rs` — `show_bottom_panel()` | Slider UI, sync decode trigger, cache rebuild on release |
| `src/cache.rs` | `SlidingWindowCache` — reference for background decode pattern |
| `src/perf.rs` | FPS measurement overlay |
| `../iced/wgpu/src/image/atlas.rs` | iced's atlas struct — persistent 2D array texture, layer management |
| `../iced/wgpu/src/image/atlas/allocator.rs` | Guillotiere 2D bin packing within 2048x2048 layers |
| `../iced/wgpu/src/image/atlas/entry.rs` | `Entry::Contiguous` vs `Entry::Fragmented` for large images |
| `../iced/wgpu/src/image/cache.rs` | iced's cache around atlas — maps image handles to atlas entries |
| `../iced/wgpu/src/image/raster.rs` | Host-side decode + upload tracking |
| `../data-viewer/src/cache/img_cache.rs` | iced viewskater's `ImageCache` — loading queue, per-pane coordination |
| `../data-viewer/src/loading_status.rs` | `LoadingStatus` queue mechanism (iced version) |
| `../data-viewer/src/navigation_keyboard.rs` | `move_right_all()`, `are_panes_cached_next()` — iced nav reference |
