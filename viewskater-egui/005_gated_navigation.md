# 005: Gated Navigation — Throttle to Cache Throughput

**Date:** 2026-03-08

## Problem

With 4K images, holding the arrow key produced bursty rendering: the user burned through 5 cached images instantly, then stalled for ~500ms while the next batch decoded. All background threads finished simultaneously (since they were spawned within milliseconds of each other), causing all 5 cells to flip green at once. Then the cycle repeated.

This differed from iced viewskater, which produces steady ~11.5 FPS during held-key navigation.

## Root cause

The egui version advanced navigation on every key repeat event regardless of cache state. On cache miss, it fell back to `load_sync()` — a blocking synchronous decode on the UI thread. This created two problems:

1. **Bursty loading**: 5 cached images consumed in ~150ms (5 × 30ms key repeat), then UI-blocking sync decodes for subsequent images
2. **Wasted preloading**: background threads for the next batch all started at nearly the same time, so they all finished together rather than staggering

## Fix: gate navigation on cache availability

Modeled after iced viewskater's `move_right_all()` in `navigation_keyboard.rs`, which checks `are_panes_cached_next()` before rendering:

```rust
// Before (bursty):
if let Some(t) = tex {
    self.current_texture = Some(t);   // cache hit
} else {
    self.load_sync(ctx);              // cache miss → blocks UI
}

// After (gated):
if let Some(t) = cache.current_texture_for(new_index) {
    self.current_index = new_index;   // only advance if cached
    self.current_texture = Some(t);
    cache.navigate_forward(new_index, &self.image_paths);
} else {
    // Don't advance — wait for background thread to finish.
    // poll() will upload the texture on a future frame,
    // and the next key repeat will find it cached.
}
```

Key behavioral change: **on cache miss, navigation is skipped rather than blocking**. The background thread is already in-flight (spawned by a previous `navigate_forward` or `initialize`). When it completes, `poll()` uploads the texture, and the next key repeat event finds it cached and proceeds.

## Why this works

The cache has 5 forward slots. Each `navigate_forward` spawns 1 new thread for the newly exposed edge. In steady state:

- Thread for position N+5 is spawned when navigating to position N
- Thread takes ~90ms to decode a 4K image
- Key repeat fires every ~30ms
- By the time the user reaches position N+5 (5 × 30ms = 150ms later), the thread has been running for 150ms > 90ms → cached

For images where decode time exceeds `cache_count × key_repeat_interval`, occasional single-frame stalls occur (instead of batch stalls). This naturally throttles navigation to decode throughput.

## What's unchanged

- `jump_to()` and slider drag still use `load_sync()` — these need immediate results since the user jumps to an arbitrary position
- `poll()` still runs every frame at the top of `update()`
- Background thread spawning and `in_flight` deduplication unchanged
- The cache overlay shows the same states; behavior difference is visible as steady green-cell filling instead of batch flipping

## Comparison with iced viewskater

| Aspect | iced viewskater | egui viewskater |
|--------|----------------|-----------------|
| Gating mechanism | `are_panes_cached_next()` check before `render_next_image_all()` | `current_texture_for()` check before advancing |
| Loading queue | `LoadingStatus` with `loading_queue` + `being_loaded_queue`, `LoadOperation` enum | `in_flight: HashSet<usize>` for deduplication |
| Thread management | `load_images_by_operation()` through iced `Task` system | Direct `std::thread::spawn` with `mpsc` channel |
| Duplicate prevention | `are_next_image_indices_in_queue()` + `is_blocking_loading_ops_in_queue()` | `in_flight.contains(&file_index)` |

The egui version is significantly simpler because: single pane (no multi-pane coordination), no `LoadOperation` enum (just spawn/poll), no priority queue (FIFO channel suffices), no CPU/GPU backend split.

## Results

4K image keyboard navigation performance:

| Version | FPS | Behavior |
|---------|-----|----------|
| Before cache (sync decode) | ~33 | Blocked UI on every navigate, limited by key repeat rate for small images |
| Cache + sync fallback | ~15-16 | Bursty — burned through cached batch, stalled, batch-loaded, repeated |
| Cache + gated navigation | **~25** | Steady — advances only when next image is cached, smooth frame pacing |
| iced viewskater (reference) | ~11.5 | Steady — same gating pattern via `are_panes_cached_next()` |

The egui version achieves ~2.2× the image FPS of the iced version on the same 4K images. Contributing factors:
- No iced message-passing overhead (event → message → update → view → diff → render)
- No `LoadOperation` queue processing or multi-pane coordination
- Direct `std::thread::spawn` vs iced `Task` system
- Simpler texture upload path (`ctx.load_texture()` vs iced's wgpu backend with optional BC1 compression)

The cache debug overlay confirms steady behavior: cells fill in one at a time from center outward as threads complete, and the current position marker advances smoothly rather than jumping in bursts.
