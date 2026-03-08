# Slider Performance: TextureHandle Reuse & LRU Decode Cache

**Date:** 2026-03-09
**Branch:** `feat/slider-performance`

## Problem

During slider drag, each position change triggers `load_sync()` — a full synchronous decode + GPU upload on the UI thread. For 4K PNG images (~10MB, ~90ms decode), this limits slider scrubbing to ~4-5 FPS. The iced version achieves ~6-7 FPS on the same images.

The bottleneck is **CPU decode**, not GPU upload. `image::open()` dominates at ~90ms per 4K PNG. GPU texture upload via egui is <5ms.

## Optimization 1: TextureHandle::set() Reuse

**Commit:** `8cefcbc`

### Before

`load_sync()` called `ctx.load_texture()` on every slider step, which allocates a **new GPU texture** each time (new `TextureId`, new `wgpu::Texture`). The previous texture is dropped and deallocated. This means every frame pays:
1. GPU deallocation of old texture
2. GPU allocation of new texture
3. Full RGBA8 pixel upload

### After

On the first call, `load_texture()` creates the initial `TextureHandle`. On subsequent calls, `TextureHandle::set(color_image, options)` replaces the pixel data in-place — same `TextureId`, no allocation/deallocation cycle. egui internally sends an `ImageDelta::full()` which overwrites the texture contents.

```rust
if let Some(tex) = &mut self.current_texture {
    tex.set(color_image, egui::TextureOptions::LINEAR);
} else {
    self.current_texture = Some(ctx.load_texture(...));
}
```

### Impact

No measurable FPS improvement — the GPU allocation overhead was small compared to the ~90ms decode bottleneck. But this is architecturally correct and eliminates unnecessary GPU churn. The real benefit compounds with the LRU cache below, where the upload path is hit much more frequently.

## Optimization 2: LRU Decode Cache

**Commit:** `8bdb956`

### Concept

Cache decoded `ColorImage` (RGBA8 pixel data) in CPU memory, keyed by file index. When the slider revisits a previously-decoded image, skip the ~90ms decode entirely and only do the GPU upload (~2-5ms via `TextureHandle::set()`).

This is the egui equivalent of iced's raster cache where `Memory::Device(entry)` returns instantly for previously-loaded images.

### Implementation

`DecodeLruCache` in `src/cache.rs`:
- `HashMap<usize, ColorImage>` for O(1) lookup by file index
- `VecDeque<usize>` tracking access order for LRU eviction
- Capacity: 50 images (configurable via `LRU_CAPACITY`)
- On `get()`: moves entry to most-recently-used position
- On `insert()`: evicts least-recently-used if at capacity

Integration in `load_sync()`:
1. Check `decode_cache.get(file_index)` — if hit, clone the `ColorImage` and upload via `tex.set()`, skip decode entirely
2. On cache miss: decode normally via `image::open()`, then `decode_cache.insert()` before uploading to GPU

### Memory budget

| Image resolution | Per-image RGBA8 size | 50 images |
|-----------------|---------------------|-----------|
| 4K (3840x2160) | ~32 MB | ~1.6 GB |
| 1080p (1920x1080) | ~8 MB | ~400 MB |

### Performance results (4K PNG)

| Scenario | FPS |
|----------|-----|
| Initial slider drag (cold cache) | ~5.5 |
| Revisiting cached images (warm cache) | ~18 |
| Iced reference (same images) | ~6-7 |

The warm cache FPS significantly exceeds iced's slider performance because revisits skip decode entirely — only the GPU upload cost remains. During typical back-and-forth scrubbing, the cache warms up quickly and most positions are hits.

## Architecture

```
slider.changed()
  → check SlidingWindowCache (keyboard nav cache, free hit)
  → if miss: check SliderLoader throttle (10ms gate)
    → if throttle passes: load_sync()
      → check DecodeLruCache (skip decode on revisit)
      → if LRU miss: image::open() → decode → insert into LRU → tex.set()
      → if LRU hit: clone ColorImage → tex.set()

slider.drag_stopped()
  → cache.jump_to() — rebuild SlidingWindowCache around final position
```

Three layers of caching, each addressing a different scenario:
1. **SlidingWindowCache** — pre-decoded neighbors for keyboard navigation (background threads)
2. **SliderLoader throttle** — rate-limits sync decodes to prevent queueing redundant work during fast drag
3. **DecodeLruCache** — CPU-side pixel cache for slider revisits, eliminates decode on warm hits

## Key files

| File | Role |
|------|------|
| `src/cache.rs` — `DecodeLruCache` | LRU cache struct with get/insert/eviction |
| `src/cache.rs` — `SliderLoader` | 10ms throttle gate |
| `src/main.rs` — `PaneState::load_sync()` | LRU check → decode → cache insert → GPU upload |
| `src/main.rs` — `show_bottom_panel()` | Slider UI, cache check chain |

## Per-step profiling: egui load_sync() (4K PNG, 3840x2160)

Instrumented each step of `load_sync()` to identify where the cold-cache frame time goes. Data from a full left-to-right slider drag over 140 images:

```
load_sync [19] (3840x2160): decode=88.2ms rgba=74.4ms convert=29.5ms cache=19.5ms upload=0.0ms total=211.5ms
load_sync [29] (3840x2160): decode=87.0ms rgba=59.0ms convert=13.0ms cache=18.2ms upload=0.0ms total=177.2ms
load_sync [36] (3840x2160): decode=77.5ms rgba=60.2ms convert=12.9ms cache=19.1ms upload=0.0ms total=169.6ms
load_sync [51] (3840x2160): decode=76.0ms rgba=58.2ms convert=12.9ms cache=18.6ms upload=0.0ms total=165.7ms
load_sync [80] (3840x2160): decode=78.0ms rgba=66.1ms convert=14.2ms cache=20.0ms upload=0.0ms total=178.2ms
load_sync [123](3840x2160): decode=65.4ms rgba=58.1ms convert=12.8ms cache=17.9ms upload=0.0ms total=154.1ms
```

### Breakdown (steady-state averages for 4K)

| Step | Code | Avg time | % of total |
|------|------|----------|------------|
| `decode` | `image::open(path)` | ~75ms | 44% |
| `rgba` | `.to_rgba8()` | ~60ms | 35% |
| `convert` | `ColorImage::from_rgba_unmultiplied()` | ~13ms | 8% |
| `cache` | `color_image.clone()` + LRU insert | ~19ms | 11% |
| `upload` | `tex.set()` | ~0ms | 0% |
| **total** | | **~170ms** | **~5.9 FPS** |

### Key observations

- **`decode` + `rgba` dominate** at ~135ms combined (79%). `image::open()` decompresses PNG, then `.to_rgba8()` converts the internal representation to RGBA8 pixels.
- **`convert` (13ms)** is `ColorImage::from_rgba_unmultiplied` — iterates every pixel (8.3M for 4K) to create egui's `Color32` structs with alpha premultiplication. Iced skips this entirely — it uploads raw RGBA bytes directly to the GPU.
- **`cache` (19ms)** is the `color_image.clone()` for LRU insert — a 32MB memcpy. On cold-cache left-to-right slides, every clone is wasted since those entries are never revisited during the same drag.
- **`upload` (~0ms)** — `tex.set()` just queues an `ImageDelta`; actual GPU upload happens later in egui's rendering pipeline and isn't captured here.

### Gap vs iced

Iced achieves ~6.6 FPS on the same 4K images = ~151ms per frame. Our 170ms total gives ~5.9 FPS. The ~19ms gap matches the `cache` clone cost almost exactly. The `convert` step (13ms) is additional overhead iced doesn't pay.

## Per-step profiling: iced framework (4K PNG, 3840x2160)

Added per-step timing to the local iced fork (`../iced/`) and ran with the same 4K images. Switched `../data-viewer/Cargo.toml` to use local path dependencies for all iced crates.

Run with: `cd ../data-viewer && RUST_LOG=iced_graphics=info,iced_wgpu=info,viewskater=info cargo run --profile opt-dev -- <path>`

### Instrumentation locations

- `iced/graphics/src/image.rs` — `load()`: times `image::load_from_memory()`, EXIF read, `.into_rgba8()`, bytes conversion
- `iced/wgpu/src/image/raster.rs` — `upload()`: times `load()` vs atlas upload

### Raw data (slider drag, 4K PNG from_bytes path)

```
[iced decode] from_bytes (3840x2160): decode=58.9ms exif=0.2ms rgba=24.8ms bytes=0.0ms total=84.0ms
[iced raster] upload (3840x2160): load=0.0ms atlas_upload=56.9ms total=56.9ms

[iced decode] from_bytes (3840x2160): decode=61.5ms exif=0.2ms rgba=21.1ms bytes=0.0ms total=82.8ms
[iced raster] upload (3840x2160): load=0.0ms atlas_upload=60.1ms total=60.1ms

[iced decode] from_bytes (3840x2160): decode=60.1ms exif=0.3ms rgba=11.9ms bytes=0.0ms total=72.3ms
[iced raster] upload (3840x2160): load=0.0ms atlas_upload=58.1ms total=58.1ms

[iced decode] from_bytes (3840x2160): decode=57.7ms exif=0.2ms rgba=11.4ms bytes=0.0ms total=69.4ms
[iced raster] upload (3840x2160): load=0.0ms atlas_upload=57.6ms total=57.6ms

[iced decode] from_bytes (3840x2160): decode=56.6ms exif=0.2ms rgba=11.3ms bytes=0.0ms total=68.1ms
[iced raster] upload (3840x2160): load=0.0ms atlas_upload=58.9ms total=58.9ms

[iced decode] from_bytes (3840x2160): decode=56.4ms exif=0.2ms rgba=11.2ms bytes=0.0ms total=67.7ms
[iced raster] upload (3840x2160): load=0.0ms atlas_upload=57.7ms total=57.8ms

[iced decode] from_bytes (3840x2160): decode=52.1ms exif=0.2ms rgba=11.4ms bytes=0.0ms total=63.7ms
[iced raster] upload (3840x2160): load=0.0ms atlas_upload=55.8ms total=55.8ms

[iced decode] from_bytes (3840x2160): decode=51.6ms exif=0.2ms rgba=11.5ms bytes=0.0ms total=63.3ms
[iced raster] upload (3840x2160): load=0.0ms atlas_upload=57.3ms total=57.3ms
```

Note: `load=0.0ms` in the raster upload because `raster::Cache::load()` is called first for dimensions during the prepare phase (which does the full decode), and `upload()` then calls `self.load()` again which hits the cache immediately.

### Iced breakdown (steady-state averages for 4K)

| Step | Code | Avg time |
|------|------|----------|
| `decode` | `image::load_from_memory(bytes)` | ~57ms |
| `exif` | EXIF orientation read | ~0.2ms |
| `rgba` | `.into_rgba8()` | ~11.5ms |
| `bytes` | `.into_raw()` + `Bytes::from()` | ~0ms |
| **decode total** | | **~69ms** |
| `atlas_upload` | `atlas.upload()` (`copy_buffer_to_texture`) | ~57ms |

Frame-to-frame intervals from timestamps: ~148-160ms average → **~6.6 FPS**.

## Side-by-side comparison: egui vs iced (4K PNG cold cache)

| Step | egui | iced | Delta |
|------|------|------|-------|
| Image decode | `image::open(path)` ~75ms | `image::load_from_memory(bytes)` ~57ms | +18ms |
| RGBA conversion | `.to_rgba8()` ~60ms | `.into_rgba8()` ~11.5ms | **+48.5ms** |
| Pixel format conversion | `from_rgba_unmultiplied` ~13ms | *(none — raw bytes to GPU)* | +13ms |
| LRU cache insert | `clone()` + insert ~19ms | *(none)* | +19ms |
| GPU upload | `tex.set()` ~0ms (deferred) | `atlas.upload()` ~57ms | -57ms |
| **Total measured** | **~170ms** | **~69ms + ~57ms = ~126ms** | |
| **Frame-to-frame** | **~170ms (~5.9 FPS)** | **~150ms (~6.6 FPS)** | **+20ms** |

### Key findings

**1. `.to_rgba8()` vs `.into_rgba8()` — 48.5ms difference (biggest single factor)**

`.to_rgba8()` allocates a new buffer and copies every pixel regardless of the source format. `.into_rgba8()` consumes the `DynamicImage` and reuses the internal buffer when it's already RGBA8 — effectively a type cast with no data copy. For PNG images that decode natively as RGBA8, `into_rgba8()` is nearly free while `to_rgba8()` copies the entire 32MB buffer.

Our code calls `.to_rgba8()` but never uses `img` afterward — switching to `.into_rgba8()` is a one-line fix that would save ~48.5ms per frame.

**2. `image::open(path)` vs `load_from_memory(bytes)` — 18ms difference**

Iced's async task reads file bytes from disk in a background thread before the decode phase. When the prepare phase calls `load_from_memory(bytes)`, it only decompresses — no file I/O. Our `image::open(path)` does both disk read + decompression on the UI thread, adding ~18ms of file I/O overhead.

**3. `from_rgba_unmultiplied` — 13ms overhead unique to egui**

egui's `ColorImage` stores `Vec<Color32>` where each `Color32` is created by `from_rgba_unmultiplied()`, which iterates every pixel (8.3M for 4K) doing alpha premultiplication. Iced uploads raw `&[u8]` RGBA bytes directly to the GPU via `copy_buffer_to_texture` with no per-pixel processing on the CPU side.

**4. LRU clone — 19ms overhead on cold cache**

The `color_image.clone()` for LRU cache insert copies the entire 32MB `ColorImage`. On cold-cache slides (one direction, no revisits), every clone is wasted work. On warm-cache revisits this pays for itself by skipping the ~69ms decode.

**5. GPU upload timing difference**

egui's `tex.set()` shows ~0ms because it only queues an `ImageDelta` — actual GPU upload happens later in the render pipeline and isn't captured. Iced's `atlas.upload()` at ~57ms records the `copy_buffer_to_texture` DMA command. The actual GPU work in both cases is similar; the difference is where in the pipeline it's measured.

## Optimization 3: Bypass CICP pipeline + opt-level fix

**Commit:** `557b27c`

### Root cause: image crate v0.25 CICP color space conversion

Investigation revealed the `rgba` step (~60ms) was not caused by `to_rgba8()` vs `into_rgba8()` — the test PNGs are RGB (no alpha), so both must convert. The actual bottleneck was image crate v0.25's new **CICP color space metadata handling**.

v0.24 (used by iced): simple direct pixel copy via `FromColor` trait:
```rust
// v0.24 — direct u8-to-u8 copy, no color space math
rgba[0] = T::from_primitive(rgb[0]);  // identity for u8→u8
rgba[1] = T::from_primitive(rgb[1]);
rgba[2] = T::from_primitive(rgb[2]);
rgba[3] = T::DEFAULT_MAX_VALUE;       // 255
```

v0.25: dispatches through `cast_in_color_space()` → `cast_pixels()`. When PNGs have sRGB/BT.709 metadata (most do), the fast path fails and falls back to `cast_pixels_by_fallback()` which:
1. Expands every pixel to f32
2. Processes in chunks of 256 through the color space transform pipeline
3. Converts back to u8

This f32 fallback costs ~60ms for 4K (8.3M pixels) vs ~11.5ms for v0.24's direct path.

### Attempted fix 1: `.to_rgba8()` → `.into_rgba8()`

No improvement. The PNGs are RGB format (confirmed via PIL: `Mode: RGB`), so `into_rgba8()` still needs to allocate a new 4-channel buffer and convert every pixel. The optimization only helps when images are already RGBA8 internally.

### Attempted fix 2: Manual RGB→RGBA byte loop

Bypassed the image crate's conversion with a manual loop:
```rust
for chunk in rgb.chunks_exact(3) {
    rgba.push(chunk[0]); rgba.push(chunk[1]); rgba.push(chunk[2]); rgba.push(255);
}
```
Reduced `rgba` from ~60ms to ~37ms, but still had the separate `from_rgba_unmultiplied` step (~17ms). Net improvement was modest and introduced an unnecessary intermediate `Vec<u8>` buffer.

### Final fix: Direct RGB → Color32 conversion

Eliminated both the RGBA intermediate buffer and `from_rgba_unmultiplied` by converting directly from decoded pixel data to egui's `Vec<Color32>`:

```rust
fn image_to_color_image(img: image::DynamicImage) -> egui::ColorImage {
    match img {
        DynamicImage::ImageRgb8(buf) => {
            let rgb = buf.into_raw();
            let pixels: Vec<egui::Color32> = rgb
                .chunks_exact(3)
                .map(|c| egui::Color32::from_rgb(c[0], c[1], c[2]))
                .collect();
            egui::ColorImage { size: [w, h], pixels }
        }
        DynamicImage::ImageRgba8(buf) => { /* similar for RGBA */ }
        other => { /* fallback via into_rgba8() for rare formats */ }
    }
}
```

This collapses what was three separate steps (`to_rgba8` → `from_rgba_unmultiplied` → overhead) into one pass.

### Profile mismatch fix

Discovered the `opt-dev` profiles were different:

| | viewskater-egui (was) | data-viewer (iced) |
|---|---|---|
| `inherits` | `dev` | `release` |
| `opt-level` | **1** | **3** |

All prior benchmarks were comparing opt-level 1 against opt-level 3. Fixed to match iced's profile: `inherits = "release"`, `opt-level = 3`.

### Performance results (4K PNG, cold cache)

| Metric | Before (opt 3, v0.25 CICP) | After (opt 3, direct Color32) | Iced reference |
|--------|---------------------------|-------------------------------|----------------|
| `convert` | ~60ms + ~13ms = ~73ms | ~13ms | ~11.5ms |
| `cache` | ~19ms | ~30ms | N/A |
| `decode` | ~75ms | ~76ms | ~57ms |
| **total** | ~170ms | ~120ms | ~150ms frame |
| **FPS** | ~5.9 | **~6.5-7** | ~6.6 |

Raw data (steady-state, opt-level 3, direct Color32):
```
load_sync [56] (3840x2160): decode=77.4ms convert=9.4ms cache=26.6ms upload=0.0ms total=113.4ms
load_sync [59] (3840x2160): decode=77.2ms convert=10.6ms cache=32.2ms upload=0.0ms total=119.9ms
load_sync [100](3840x2160): decode=72.3ms convert=12.3ms cache=26.5ms upload=0.0ms total=111.2ms
load_sync [127](3840x2160): decode=66.1ms convert=10.6ms cache=28.6ms upload=0.0ms total=105.3ms
load_sync [134](3840x2160): decode=62.4ms convert=12.3ms cache=30.4ms upload=0.0ms total=105.1ms
```

### Updated breakdown (steady-state averages, 4K, opt-level 3)

| Step | Code | Avg time | % of total |
|------|------|----------|------------|
| `decode` | `image::open(path)` | ~76ms | 63% |
| `convert` | `image_to_color_image()` (direct RGB→Color32) | ~13ms | 11% |
| `cache` | `color_image.clone()` + LRU insert | ~30ms | 25% |
| `upload` | `tex.set()` | ~0ms | 0% |
| **total** | | **~120ms** | **~8.3 FPS theoretical** |

Note: measured total ~120ms suggests theoretical ~8.3 FPS, but actual FPS is ~6.5-7 due to egui frame overhead (layout, painting, present) not captured in `load_sync()` timing.

## 1080p results

With all optimizations applied (direct Color32, opt-level 3, LRU cache):

| Scenario | egui | iced |
|----------|------|------|
| 1080p slider drag | **~30 FPS** | ~25 FPS |
| 4K slider drag (cold) | ~6.5-7 FPS | ~6.6 FPS |
| 4K slider drag (warm cache) | ~18 FPS | ~6.6 FPS |

egui now exceeds iced's slider performance at both resolutions.

## Why egui's default API forces a double-pass

egui's `ColorImage` stores `Vec<Color32>` but its only byte-based constructor is `ColorImage::from_rgba_unmultiplied(size: [usize; 2], rgba: &[u8])`. This means the natural usage pattern forces two full pixel iterations:

1. Convert decoded image to RGBA bytes: `img.to_rgba8().into_raw()` → `Vec<u8>` (pass 1)
2. Feed bytes to egui: `ColorImage::from_rgba_unmultiplied(size, &bytes)` → iterates every pixel to build `Vec<Color32>` (pass 2)

There's no `from_rgb()` constructor, no way to pass `Vec<Color32>` directly through an API, and no way to feed raw RGB bytes without an intermediate RGBA buffer. For small UI textures this overhead is negligible. For a full-screen image viewer pushing 8.3M pixels per frame, two full iterations plus v0.25's CICP color space overhead turns a ~13ms operation into ~73ms.

Our fix sidesteps this by constructing `ColorImage { size, pixels }` directly — the struct fields are `pub`, so we build the `Vec<Color32>` ourselves in a single pass from the decoded RGB data without any intermediate buffer or second iteration.

## Remaining optimization opportunities

1. **Eliminate cold-cache clone** — `cache` at ~30ms is now 25% of the total. Skip LRU insert during active drag, or use `Arc<ColorImage>` (needs egui fork to accept `Arc<ColorImage>` in `tex.set()`).
2. **Separate file I/O from decode** — `image::open()` includes disk read. Iced reads bytes asynchronously and decodes via `load_from_memory()` (57ms vs 76ms = ~19ms file I/O overhead).
3. **Faster decode library** — `zune-png`, `libpng` FFI, or half-resolution decode during slider drag.
4. **Predictive decode** — spawn background threads to decode ahead of slider drag direction.
5. **Downscaled slider preview** — decode at half or quarter resolution during drag.
