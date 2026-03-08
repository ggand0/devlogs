# Slider Image Rendering: Iced Atlas Analysis & egui Strategy

**Date:** 2026-03-08
**Goal:** Analyze how the iced viewskater + iced backend render slider images via the atlas, and derive the right optimization strategy for the egui port.

## Iced Viewskater — Slider Flow (App Side)

The iced version uses a **dual-path strategy** for slider drag vs release:

### During Slider Drag (fast path)

```
DualSlider::on_event() — CursorMoved during drag
  → on_change callback fires
  → Message::SliderChanged(pane_index, value)
```

**SliderChanged handler** (`message_handlers.rs:453-521`):
1. Sets `app.is_slider_moving = true`, `app.use_slider_image_for_render = true`
2. Determines async/throttle flags:
   - `use_async = true` (always during slider drag)
   - `use_throttle = true` on Linux only (`cfg(target_os = "linux")`)
3. Calls `navigation_slider::update_pos(panes, pane_index, pos, use_async=true, use_throttle)`

**update_pos** (`navigation_slider.rs:501-638`):
1. Stores position in `LATEST_SLIDER_POS` (global AtomicUsize)
2. If throttle enabled: checks `LAST_SLIDER_LOAD` (Lazy<Mutex<Instant>>)
   - Linux threshold: 10ms
   - If elapsed < threshold → returns `Task::none()`, skipping all loading
3. Spawns async image load task per pane:
   - `create_async_image_widget_task()` → reads image bytes from disk → creates `image::Handle::from_bytes(bytes)` (no decode yet)
   - Returns `Message::SliderImageWidgetLoaded(pane_idx, pos, handle, dimensions, file_size)`

**SliderImageWidgetLoaded handler** (`message_handlers.rs:392-419`):
- Stores `pane.slider_image = Some(handle)` for display
- Does NOT update `current_index` (prevents desync with stale async results)

**UI render** (`ui.rs:545-750`):
- If `pane.slider_image` exists → renders via iced's `Image` widget (the handle gets decoded + atlas-uploaded by iced's engine prepare phase)
- Otherwise → falls back to shader-based full image

### After Slider Release (full-res path)

```
DualSlider::on_event() — ButtonReleased
  → on_release callback fires
  → Message::SliderReleased(pane_index, value)
```

**SliderReleased handler** (`message_handlers.rs:523-566`):
1. `app.is_slider_moving = false`
2. Calls `navigation_slider::load_remaining_images()`:
   - Sync-loads full-res image at final position
   - Enqueues async neighbor loads via `get_loading_tasks_slider()`
   - Neighbors loaded in parallel via `load_images_async()` → `Message::ImagesLoaded`

## Iced Engine — Atlas Rendering Pipeline (Framework Side)

When an `Image` widget displays a `Handle`, the iced engine processes it through:

### Accumulation Phase (draw)

```
Viewer::draw()
  → renderer.draw_image(Image { handle, filter_method, rotation, opacity }, bounds)
  → wgpu Renderer::draw_image()
  → Layer::draw_raster(image, bounds, transformation)
  → layer.images.push(Image::Raster(image, transformed_bounds))
```

Images are accumulated in batches per layer during the draw traversal. No GPU work happens here.

### Prepare Phase (once per frame)

```
Engine::prepare()
  → Pipeline::prepare(device, encoder, belt, cache, images, projection, scale)
  → for each Image in batch:
      → cache.upload_raster(device, encoder, &handle)
      → raster cache lookup by handle.id()

        CACHE HIT: Memory::Device(entry) → return &entry     [0ms]
        CACHE MISS:
          → load(handle)                                       [~90ms for 4K PNG]
            → image::open(path) or load_from_memory(bytes)
            → convert to RGBA8
          → atlas.upload(device, encoder, w, h, rgba_data)
            → allocate(w, h) → Entry (Contiguous or Fragmented)
            → grow() if new layers needed (copies old layers to larger texture)
            → upload_allocation() → copy_buffer_to_texture (DMA)
          → *memory = Memory::Device(entry)  // cached for future frames

      → add_instances(entry) → Instance {
          position, center, size,        // screen space
          position_in_atlas, size_in_atlas,  // normalized UV coords
          layer,                         // texture array layer index
          rotation, opacity, snap
        }
```

### Render Phase

```
Pipeline::render(cache, layer, bounds, render_pass)
  → render_pass.set_pipeline(&self.pipeline)
  → render_pass.set_bind_group(1, atlas.bind_group())   // entire texture array
  → render_pass.set_vertex_buffer(0, instances.slice(..))
  → render_pass.draw(0..6, 0..instance_count)           // instanced quad draw
```

### Shader (image.wgsl)

```wgsl
// Vertex: map quad vertex to atlas UV coordinates
out.uv = v_pos * input.atlas_scale + input.atlas_pos;
out.layer = f32(input.layer);

// Fragment: sample from 2D texture array
return textureSample(u_texture, u_sampler, input.uv, i32(input.layer))
       * vec4(1.0, 1.0, 1.0, input.opacity);
```

Switching displayed image = updating instance data (atlas UV + layer index). Sub-millisecond. No texture re-upload.

## Atlas Data Structures

```
Atlas
  ├─ texture: wgpu::Texture           2048x2048 x N layers (2D array)
  ├─ texture_view: D2Array view
  ├─ texture_bind_group
  └─ layers: Vec<Layer>
      ├─ Empty
      ├─ Busy(Allocator)               guillotiere 2D bin packing
      └─ Full                          entire 2048x2048 layer occupied

Entry (allocation result)
  ├─ Contiguous(Allocation)
  │   ├─ Partial { layer, region }     packed into a layer by guillotiere
  │   └─ Full { layer }               occupies entire layer
  └─ Fragmented { size, fragments }    images > 2048px split across layers
      └─ Fragment { position: (x,y), allocation }
```

**Growth**: when all layers are full, a new wgpu::Texture is created with +1 layer, and all existing layers are copied via `copy_texture_to_texture`. This is the one stall point.

**Fragment uploads are NOT parallelized** — each fragment's `copy_buffer_to_texture` is recorded sequentially into the same CommandEncoder. The GPU may batch the DMA operations internally when the command buffer is submitted.

## Key Insight: Why Iced's Slider is Actually Faster

The atlas is **not the primary reason** the iced slider is faster than egui's. Three things contribute:

### 1. Async loading during drag (biggest factor)

Iced spawns async tasks for each slider position. The UI thread never blocks on decode. In contrast, egui's `load_sync()` does a full synchronous decode (~90ms for 4K PNG) on the UI thread every slider step.

### 2. Raster cache hits on revisit

When revisiting an image, iced's raster cache returns `Memory::Device(entry)` instantly — the texture is already in the atlas. This matters during back-and-forth scrubbing. Same benefit an LRU TextureHandle cache would provide in egui.

### 3. Linux-only throttling

On Linux, iced throttles slider updates to 10ms minimum intervals, preventing the decode pipeline from being overwhelmed. macOS/Windows have no throttling. The throttle is in `update_pos()` using a global `LAST_SLIDER_LOAD: Lazy<Mutex<Instant>>`.

### What the atlas actually optimizes

The atlas optimizes **GPU memory layout and upload efficiency**, not decode speed:
- Bin-packing multiple images into shared 2048x2048 layers (memory efficiency)
- Incremental `copy_buffer_to_texture` into existing texture (no allocation churn)
- Instanced rendering from a single bind group (draw call efficiency)
- Optional BC1 compression (4x GPU memory reduction)

For a single full-screen image viewer, these GPU-side optimizations have marginal impact compared to the ~90ms CPU decode bottleneck.

## egui's TextureHandle::set() Behavior

Investigation confirmed:
- `TextureHandle::set()` **keeps the same TextureId** — no new GPU allocation
- Sends `ImageDelta::full()` which replaces texture contents in-place
- `set_partial()` also exists for sub-region updates
- The `TextureManager` uses reference counting; textures freed when all `TextureHandle` clones are dropped

This means texture reuse in egui is nearly free — `set()` avoids the allocation churn of creating a new texture per `load_sync()` call.

## Optimization Strategy for egui

### What to implement

1. **Async slider loading** — spawn background decode tasks during slider drag instead of blocking with `load_sync()`. This is the biggest win and directly mirrors iced's approach.

2. **LRU texture cache** — retain TextureHandles for recently-viewed images. Cache hit = 0ms (same as iced's raster cache + atlas). This is architecturally the same layer as iced's `cache.rs`.

3. **TextureHandle reuse** — use `set()` for sync fallback to avoid GPU allocation churn.

4. **Optional throttling** — rate-limit slider update processing (10ms on Linux, matching iced).

### Future extensibility to full atlas

The LRU cache is the same abstraction layer as iced's `cache.rs` — it manages which images are retained and handles eviction policy. If a full atlas is needed later:
- The caching policy layer (LRU eviction, capacity management) carries over directly
- The background decode infrastructure (threads, mpsc, decode results) carries over directly
- The texture storage layer would change: individual TextureHandles → atlas entries (layer + UV rect)
- The rendering path would change: `painter.image(tex.id())` → custom shader sampling from texture array

## Performance Targets

| Scenario | Current egui | Iced reference | Target |
|----------|-------------|----------------|--------|
| 4K PNG slider drag | ~4-5 FPS | ~7-8 FPS | >= 8 FPS |
| 1080p slider drag | ~15 FPS | ~20 FPS | >= 20 FPS |
| 4K keyboard nav | ~25 FPS | ~11.5 FPS | Maintain |

## Key Source Paths

### egui viewskater (this repo)
| File | Role |
|------|------|
| `src/main.rs` — `PaneState::load_sync()` | Current slider decode (sync, blocking) |
| `src/main.rs` — `show_bottom_panel()` | Slider UI, triggers sync decode on change |
| `src/cache.rs` | `SlidingWindowCache` — background decode for keyboard nav |
| `src/perf.rs` | FPS measurement overlay |

### iced viewskater (reference app)
| File | Role |
|------|------|
| `../data-viewer/src/widgets/dualslider.rs` | Slider widget, fires on_change/on_release |
| `../data-viewer/src/app/message_handlers.rs:453-566` | SliderChanged/Released handlers |
| `../data-viewer/src/navigation_slider.rs:501-638` | `update_pos()` — async load + throttling |
| `../data-viewer/src/navigation_slider.rs:421-499` | `create_async_image_widget_task()` |
| `../data-viewer/src/cache/img_cache.rs` | Image cache, loading queue |

### iced engine (framework)
| File | Role |
|------|------|
| `../iced/widget/src/image/viewer.rs` | Viewer widget, `draw()` → `renderer.draw_image()` |
| `../iced/wgpu/src/image/mod.rs` | Image pipeline — prepare, add_instances, render |
| `../iced/wgpu/src/image/cache.rs` | Cache wrapper — maps handles to atlas entries |
| `../iced/wgpu/src/image/raster.rs` | Raster cache — Host/Device memory states |
| `../iced/wgpu/src/image/atlas.rs` | Atlas — persistent 2D array texture, upload, grow |
| `../iced/wgpu/src/image/atlas/allocator.rs` | Guillotiere 2D bin packing |
| `../iced/wgpu/src/image/atlas/entry.rs` | Entry::Contiguous vs Entry::Fragmented |
| `../iced/wgpu/src/shader/image.wgsl` | Vertex/fragment shader — atlas UV sampling |
