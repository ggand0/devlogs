# 002: Image Rendering FPS Display

**Date:** 2026-03-07

## Goal

Add a performance overlay that measures image rendering throughput — how many unique images are displayed per second — to establish a baseline before implementing the sliding window cache.

## Key distinction: image FPS vs UI FPS

egui in reactive mode repaints at vsync (60-144+ Hz) whenever there's input. Counting `update()` calls gives you the **UI event loop FPS**, which stays high even if you're displaying the same image. This is useless for evaluating image rendering performance.

What matters is **image rendering FPS**: how many unique images are decoded, uploaded, and displayed per second. This is the metric viewskater's `ImageDisplayTracker` (in the iced fork at `iced/wgpu/src/image/mod.rs`) uses — it records a timestamp each time a new image is uploaded to the GPU, then computes FPS from the rolling window of those timestamps.

## Implementation

### `src/perf.rs` — `ImagePerfTracker`

Extracted to its own module. Tracks two things:

1. **Image timestamps** (`VecDeque<Instant>`) — recorded in `load_current()` each time a new image is successfully decoded and uploaded via `ctx.load_texture()`. Not recorded on every frame.

2. **Last decode time** (`Option<f64>`) — wall-clock time for `image::open()` + `to_rgba8()` + `ctx.load_texture()`.

### FPS calculation

Uses the same formula as viewskater's `ImageDisplayTracker::calculate_fps()`:

```rust
fn image_fps(&mut self) -> f64 {
    // Prune timestamps older than 2 seconds
    let cutoff = Instant::now() - Duration::from_secs(2);
    while self.image_timestamps.front().is_some_and(|t| *t < cutoff) {
        self.image_timestamps.pop_front();
    }
    // FPS = (N-1) / time_span_of_N_timestamps
    let span = newest.duration_since(oldest).as_secs_f64();
    (self.image_timestamps.len() - 1) as f64 / span
}
```

The 2-second window means:
- When actively navigating, FPS reflects actual throughput (e.g., ~30 images/sec if each decode takes ~33ms)
- When idle, old timestamps expire and FPS decays to 0
- Single image loads briefly show as FPS before decaying

### Overlay rendering

Uses `egui::Window` anchored to top-right, non-interactable, with semi-transparent dark background:
```
Img: 28.5 FPS | Decode: 12.3ms
```

### Integration with App

`load_current()` records the event:
```rust
let decode_ms = start.elapsed().as_secs_f64() * 1000.0;
self.perf.record_image_load(decode_ms);
```

`update()` draws the overlay:
```rust
self.perf.show_overlay(ctx);
```

## Baseline measurements (pre-cache)

With synchronous decoding (no cache), the image FPS during keyboard navigation is limited by decode time. For typical images:
- Small (640x480 PNG): decode ~2-5ms → ~30+ img/s (limited by OS key repeat rate)
- Medium (1920x1080 PNG): decode ~15-30ms → ~30+ img/s (still key-repeat limited)
- Large (4K PNG): decode ~50-100ms → FPS drops visibly below key repeat rate

After the sliding window cache is added, cached navigation should show near-zero decode time and FPS limited only by the key repeat / frame rate.
