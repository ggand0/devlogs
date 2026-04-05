# 019 Memory usage display & investigation

Branch: `fix/memory-usage`

## Summary

Added process memory (RSS) display to the existing FPS overlay using the `sysinfo` crate. Investigated and fixed high memory usage during slider scrubbing of 4K image sequences by switching the decode LRU cache from a fixed entry count to a memory budget.

## Memory display

Added RSS tracking to `ImagePerfTracker` in `src/perf.rs`. The `sysinfo` crate queries the current process's memory, throttled to once per second to avoid overhead. Display format: `Img: 12.3 FPS | 234 MB` (or GB if >= 1 GB). Appears in both the menu bar and fullscreen overlay, reusing the existing `fps_text()` path — no changes needed to `app.rs` or `menu.rs`.

## Memory investigation

### Observed behavior (before fix)

Test data: `4k_PNG_10MB` — 140 frames, 3840×2160 PNG, ~10 MB each on disk.

| State                       | RSS      |
|-----------------------------|----------|
| App startup                 | 79 MB    |
| After scrubbing slider      | 2.3 GB   |
| After closing directory     | 889 MB   |
| After opening new directory | ~889 MB  |

### Root cause: decoded image size vs file size

A 10 MB compressed PNG on disk becomes **31.6 MB** decoded as RGBA pixels in memory:

```
3840 × 2160 × 4 bytes/pixel = 33,177,600 bytes ≈ 31.6 MB
```

### Memory breakdown (before fix, steady-state)

| Component                  | Calculation              | Size     |
|----------------------------|--------------------------|----------|
| DecodeLruCache (50 entries)| 50 × 31.6 MB            | 1,580 MB |
| SlidingWindowCache (11 slots) | 11 × 31.6 MB         | 348 MB   |
| **Subtotal**               |                          | **1,928 MB** |
| Allocator overhead + fragmentation |                   | ~370 MB  |
| **Observed total**         |                          | **~2,300 MB** |

The `DecodeLruCache` (default capacity 50) stores decoded `ColorImage` objects in CPU RAM. This is the dominant consumer.

### Transient allocations during decode

Each `load_sync()` call creates transient memory pressure beyond steady-state:

1. `image::open()` decompresses PNG into `DynamicImage` (RGB8): **~24.9 MB**
2. `image_to_color_image()` converts to `ColorImage` (RGBA `Vec<Color32>`): **~31.6 MB**
   - Uses `into_raw()` so the RGB buffer is consumed, not copied
   - Brief overlap during `.collect()`: ~56.5 MB peak
3. `color_image.clone()` for LRU cache insert: **~31.6 MB** transient copy
4. Original goes to `tex.set()` / `ctx.load_texture()` for GPU upload

These don't accumulate — each is freed within the same `load_sync()` call. The `DynamicImage` intermediate is not a problem since `into_raw()` transfers ownership of the pixel buffer (zero-copy), and only one decode is in flight at a time. But rapid slider scrubbing creates many of these transient peaks, contributing to allocator fragmentation.

### Why memory stays high after close

`pane.close()` properly clears everything:
- `decode_cache.clear()` — drops all ColorImage entries
- `cache = None` — drops all TextureHandles
- `current_texture = None`

Two layers of memory retention:

**1. glibc malloc arena retention (CPU heap):** The LRU decode cache's `ColorImage` data is freed from Rust's perspective, but glibc keeps the heap arena expanded for reuse. Fixed with `libc::malloc_trim(0)` after close.

Note: a deferred trim approach (waiting 3 frames for GL texture deletion before trimming) was attempted but dropped — the residual memory after close is the GL driver pool, not the glibc heap, so trim timing made no measurable difference.

**2. OpenGL driver memory pool (GPU/shared memory):** After `glDeleteTextures` runs, the GL driver (Mesa on Linux, Metal-backed GL on macOS) keeps its internal memory pool allocated for reuse rather than returning pages to the OS. This is documented driver behavior — the OpenGL spec only guarantees the texture name is freed and contents are gone, but says nothing about physical memory. Drivers intentionally pool freed GPU memory because allocation/deallocation is expensive. There is no GL API to force the driver to release the pool. This accounts for ~570 MB residual after close (the sliding window's 11 textures × 32 MB + driver overhead). This memory IS reused when opening a new directory.

References:
- https://community.khronos.org/t/memory-usage-after-gldeletetextures/60108
- https://community.khronos.org/t/opengl-textures-and-memory-leaks/107089
- https://community.khronos.org/t/memory-leak-when-deleting-texture/42541
- https://www.gamedev.net/forums/topic/624671-gldeletetextures-does-not-remove-picture-from-memory/4938658/

### Rendering backend

The app uses **glow (OpenGL)** via eframe's default features, not wgpu. Evidence: `cargo tree -p eframe` shows `egui_glow` and `glow` as dependencies with no `wgpu` or `egui-wgpu` present. eframe 0.31 defaults to `glow` unless the `wgpu` feature is explicitly enabled.

## Fix: memory-budget-based LRU cache

### Algorithm change

Before — entry-count based:
```
insert(image):
  if entries.len() >= 50:    // fixed count, regardless of image size
    evict oldest entry
  store image
```

After — memory-budget based:
```
insert(image):
  new_bytes = width × height × 4
  while total_bytes + new_bytes > 512 MB:   // budget in bytes
    evict oldest entry
    total_bytes -= evicted image's bytes
  total_bytes += new_bytes
  store image
```

The key difference: before, it always allowed 50 entries regardless of whether they were 8 MB (1080p) or 32 MB (4K). Now it tracks actual byte usage, so 4K images get ~16 entries and 1080p gets ~64 for the same 512 MB budget.

Equivalent value for old capacity=50 with 4K: 50 × 32 MB = **1600 MB**. Set the slider to 1600 to match the old behavior.

### Performance analysis of insert()

Before (entry-count):
```rust
pub fn insert(&mut self, file_index: usize, image: egui::ColorImage) {
    if self.entries.contains_key(&file_index) {          // 1. HashMap lookup: O(1)
        self.order.retain(|&i| i != file_index);         // 2. Linear scan of VecDeque: O(n)
    } else if self.entries.len() >= self.capacity {       // 3. Compare count vs 50: O(1)
        if let Some(evicted) = self.order.pop_front() {  // 4. Pop front: O(1)
            self.entries.remove(&evicted);                // 5. HashMap remove: O(1)
        }
    }
    self.entries.insert(file_index, image);               // 6. HashMap insert: O(1)
    self.order.push_back(file_index);                     // 7. Push back: O(1)
}
```
- Step 2 is the expensive path (O(n) linear scan), but only on duplicate inserts
- Step 3 is a simple integer comparison — at most one eviction

After (memory-budget):
```rust
pub fn insert(&mut self, file_index: usize, image: egui::ColorImage) {
    let new_bytes = Self::image_bytes(&image);            // 1. Two multiplies: O(1)

    if let Some(old) = self.entries.remove(&file_index) { // 2. HashMap remove: O(1)
        self.total_bytes -= Self::image_bytes(&old);      // 3. Two multiplies + subtract: O(1)
        self.order.retain(|&i| i != file_index);          // 4. Linear scan: O(n) (same as before)
    }

    while self.total_bytes + new_bytes > self.budget_bytes { // 5. Addition + compare: O(1) per iter
        if let Some(evicted) = self.order.pop_front() {      // 6. Pop front: O(1)
            if let Some(img) = self.entries.remove(&evicted) { // 7. HashMap remove: O(1)
                self.total_bytes -= Self::image_bytes(&img);   // 8. Two multiplies + subtract: O(1)
            }
        } else { break; }
    }

    self.total_bytes += new_bytes;                        // 9. Add: O(1)
    self.entries.insert(file_index, image);               // 10. HashMap insert: O(1)
    self.order.push_back(file_index);                     // 11. Push back: O(1)
}
```

The only new work per call is `image_bytes()` — two `usize` multiplies reading fields already in L1 cache. The eviction loop (steps 5-8) can iterate more than once if images vary in size, but for uniform-resolution sequences (like the 4K dataset) it's always exactly one eviction, same as before. The O(n) `retain` on duplicates is identical in both versions.

The eviction loop keeps evicting the oldest cached image until there's room for the new one. For mixed-resolution directories (e.g. evicting a small 1080p image to make room for a large 4K image), one eviction might not free enough bytes, so it loops. The `else { break }` handles the edge case where the order queue is empty.

This is the only code change that runs during the scrubbing hot path. The `sysinfo` memory poll is once per second (gated by `Instant` comparison), and `malloc_trim` only runs on directory close.

### Results after fix

| State                       | Before   | After      |
|-----------------------------|----------|------------|
| App startup                 | 79 MB    | 79 MB      |
| After loading directory     | 653 MB   | 653 MB     |
| After scrubbing slider (4K) | 2.3 GB   | 1.2-1.3 GB |
| After closing directory     | 889 MB   | ~645 MB    |

The ~645 MB after close is the OpenGL driver's retained memory pool — not a leak, and reused on next directory open.

### Settings change

- Renamed `lru_capacity` (entry count, default 50) → `lru_budget_mb` (MB, default 1024)
- Settings slider range: 128–4096 MB
- Old settings files: `lru_capacity` field is ignored via `#[serde(default)]`, new default of 512 MB applies

### Tradeoff

Fewer cached entries for 4K content (16 vs 50) means more cache misses during rapid scrubbing, leading to slightly lower scrub FPS. This trades scrub smoothness for bounded memory. Users can increase the budget in Preferences if they prefer the old behavior.

## Rendering backend investigation

### glow (OpenGL) vs wgpu

The app uses **glow (OpenGL)** by default via eframe's default features. Evidence: `cargo tree -p eframe` shows `egui_glow` and `glow` as dependencies with no `wgpu` or `egui-wgpu`. eframe 0.31's default features include `"glow"` but not `"wgpu"`.

Tested switching to wgpu to see if it handles texture memory differently after close. Findings:

- **Memory after close**: similar retention — wgpu also keeps GPU memory pools allocated
- **Visual quality**: wgpu applies **dithering** by default (shader-level `dither_interleaved` in `egui.wgsl`), which adds subtle per-pixel noise to reduce color banding. On photographic images this looks like very slight softness compared to glow's direct pixel output
- **Performance**: FPS values similar, but subjectively feels slightly slower with wgpu

Conclusion: glow is the better default for an image viewer where pixel crispness matters. A `--renderer` CLI flag to switch between backends is planned for a separate branch (`feat/renderer-selection`).

## Files changed

- `Cargo.toml` — added `sysinfo = "0.33"`, `libc = "0.2"`
- `src/perf.rs` — added `sysinfo` memory tracking to `ImagePerfTracker`, updated `fps_text()` to include memory
- `src/cache.rs` — `DecodeLruCache` rewritten to track total bytes and evict based on budget
- `src/settings.rs` — `lru_capacity` → `lru_budget_mb`, default 512, slider 128–4096
- `src/pane.rs` — updated field name, added total_mb to debug log
- `src/app.rs` — updated field references, added deferred trim countdown
- `src/app/handlers.rs` — updated field references, deferred `malloc_trim` by 3 frames after close
- `README.md` — added Memory usage section
