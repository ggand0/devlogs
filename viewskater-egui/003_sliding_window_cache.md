# 003: Sliding Window Cache & Async Image Loading

**Date:** 2026-03-07

## Goal

Replace synchronous image decoding on every navigation with a sliding window cache that preloads neighboring images in background threads, making keyboard navigation instant for cached images.

## Architecture

Two files changed:
- `src/cache.rs` (**new**, ~250 lines) — `SlidingWindowCache`, `DecodeResult`, all cache logic
- `src/main.rs` (modified) — cache integration into `App`

No new dependencies — uses `std::thread`, `std::sync::mpsc`, `std::collections::{VecDeque, HashSet}` from std.

## Cache design

### Window layout

`cache_count = 5`, so `cache_size = 5 * 2 + 1 = 11` slots. The `VecDeque<Option<TextureHandle>>` holds preloaded GPU textures. `first_file_index` tracks which file index maps to slot 0.

```
Example: current_index=15, cache_count=5
first_file_index = 10
slots: [10, 11, 12, 13, 14, [15], 16, 17, 18, 19, 20]
                              ^current (slot 5, center)

Example: current_index=2, cache_count=5 (near start boundary)
first_file_index = 0
slots: [0, 1, [2], 3, 4, 5, 6, 7, 8, 9, 10]
               ^current (slot 2, off-center)
```

This is simpler than viewskater's `current_offset` approach — `first_file_index` directly maps slot indices to file indices: `file_index = first_file_index + slot_index`.

### Workflow: forward navigation (right arrow key)

Step-by-step trace when user presses right arrow at `current_index=15`:

**1. Key event → `handle_keyboard()` → `navigate(1, ctx)`** (main.rs:213-214)

**2. `navigate()` computes new index** (main.rs:126-127):
```
new_index = clamp(15 + 1, 0, num_files-1) = 16
```

**3. `navigate()` calls `cache.navigate_forward(16, image_paths)`** (main.rs:138)

**4. `navigate_forward()` checks if window needs to shift** (cache.rs:130-143):
```
current_slot = 16 - 10 = 6
cache_count = 5
6 > 5 → shift window right:
  - slots.pop_front()       → drops slot for file 10
  - slots.push_back(None)   → new empty slot at end
  - first_file_index = 11   → window is now [11..21]
  - spawn_load(21)          → background thread starts decoding file 21
```

**5. `navigate_forward()` returns the cached texture** (cache.rs:145):
```
current_texture_for(16):
  slot_idx = 16 - 11 = 5   → slots[5]
  If background thread already finished → Some(TextureHandle) → cache hit
  If not yet finished → None → cache miss
```

**6. Back in `navigate()`** (main.rs:143-149):
- Cache hit → `self.current_texture = Some(t)`, records `perf.record_image_load(0.0)` (decode shows 0ms)
- Cache miss → falls back to `self.load_sync(ctx)` which blocks on `image::open()` + `ctx.load_texture()`

### Background thread lifecycle

`spawn_load()` (cache.rs:195-227):

1. Checks `in_flight` HashSet to avoid duplicate spawns for same file index
2. Spawns `std::thread::spawn` with cloned `path`, `tx` (channel sender), `ctx`
3. Thread does: `image::open()` → `to_rgba8()` → `ColorImage::from_rgba_unmultiplied()`
4. Sends `DecodeResult { file_index, image: Some(ColorImage), decode_ms }` through `mpsc` channel
5. Calls `ctx.request_repaint()` to wake the UI thread

The thread does CPU-side decoding only. GPU texture upload (`ctx.load_texture()`) must happen on the main thread.

### Poll loop

`poll()` is called at the start of every `update()` frame (main.rs:359-361):

```rust
while let Ok(result) = self.rx.try_recv() {
    self.in_flight.remove(&result.file_index);
    if let Some(slot_idx) = self.slot_index_for(result.file_index) {
        // Result is still within the current window → upload texture
        self.slots[slot_idx] = Some(self.ctx.load_texture(name, color_image, ...));
    }
    // else: window shifted past this index → silently discard
}
```

This is where `ColorImage` (CPU) becomes `TextureHandle` (GPU). The two-phase design (decode on thread, upload on main) is necessary because `ctx.load_texture()` must run on the main/render thread.

### Initialization

`initialize()` (cache.rs:57-94):
1. Drain any pending results from previous window
2. Clear `in_flight` set
3. Position window: `first_file_index = clamp(center - cache_count, 0, max_first)`
4. **Synchronously** decode center image via `decode_sync()` — user sees it immediately
5. Spawn background loads for all other 10 slots

### Jump (slider release, Home/End)

`jump_to()` delegates to `initialize()` — full cache rebuild. During slider drag, only sync decode runs (no cache thrashing). Cache rebuilds on `drag_stopped()`.

### Edge cases

- **Boundaries**: `first_file_index` clamped. Near-boundary slots beyond file count stay `None`.
- **Fast navigation**: Spawned loads that finish after the window has shifted past are discarded in `poll()` via `slot_index_for()` range check.
- **Stale threads on jump**: `initialize()` drains the rx channel. Abandoned threads run to completion but results are discarded.
- **Small directories** (fewer files than cache_size): Window covers all files, no shifting.
- **Deduplication**: `in_flight: HashSet<usize>` prevents spawning multiple threads for the same file index.

## Key difference from viewskater

viewskater's `ImageCache` uses:
- `LoadOperation` enum with `LoadCenter`/`LoadPrevious`/`LoadNext` variants
- Dual queues (center priority + neighbor queue)
- `current_offset` tracking relative to cache center
- CPU/GPU backend split with BC1 compression option

This implementation is much simpler:
- Direct `first_file_index` mapping (no offset arithmetic)
- Single `mpsc` channel (no operation queues)
- Plain `std::thread::spawn` (no thread pool)
- `ColorImage` → `TextureHandle` only (no compression)

## Performance observation

With the AirSim test images (40 PNGs, ~640x480), keyboard navigation FPS is ~33 both before and after the cache. This is because:

1. These images decode in ~2-5ms
2. OS key repeat rate is ~33Hz (~30ms between repeats)
3. The bottleneck was always the key repeat rate, not decode time

The cache benefit shows in two ways:
- **Decode time in overlay**: Should show 0.0ms on cache hits vs 2-5ms on sync decode
- **Larger images**: For 4K+ images where decode > 30ms, the cache would eliminate the decode bottleneck and keep navigation at key repeat rate

To verify the cache is working: check the "Decode" value in the FPS overlay. If it shows 0.0ms during arrow key navigation, images are being served from cache.
