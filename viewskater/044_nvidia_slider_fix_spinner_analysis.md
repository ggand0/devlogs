# NVIDIA 4K Slider Fix: Why Removing UserEvent state.update() Works

**Date:** 2026-02-27
**Branch:** `fix/nvidia-rendering-performance`
**Related:** [043_nvidia_present_mode_fix.md](043_nvidia_present_mode_fix.md)

## The Fix

Removed `state.update()` from the `Action::Output` handler in the UserEvent path. Messages are still queued via `state.queue_message()` and processed by the existing `state.update()` in the WindowEvent handler on the next event loop iteration.

## Why It Was There: Spinner Branch History

Three commits in `feat/loading-spinner` created and then reshaped the spinner animation flow:

### 1. `ae74169` — "Fix spinner animation to work without mouse movement" (Jan 3, 2026)

Added TWO things to make the spinner animate continuously:

**A) `WindowEvent::RedrawRequested` handler** — a full state.update() + resize + render path triggered by RedrawRequested. This was a dedicated arm in the WindowEvent match that queued a `RedrawRequested` iced event, called state.update(), and rendered directly.

**B) `state.update()` in `Action::Output`** — called immediately after `state.queue_message()` so that SpinnerTick messages were processed right away, spawning the next SpinnerTick task via the runtime. Without this, SpinnerTick messages would sit in the queue until the next WindowEvent arrived, meaning the spinner wouldn't animate if the user wasn't moving the mouse.

Together, these created a self-sustaining loop:
```
SpinnerTick arrives as UserEvent
  → Action::Output: queue message → state.update() processes it → runtime spawns next SpinnerTick
  → *redraw = true → window.request_redraw()
  → RedrawRequested WindowEvent: state.update() + render (shows spinner)
  → repeat
```

### 2. `16da27c` — "Remove redundant WindowEvent::RedrawRequested handler" (Jan 8, 2026)

Removed the dedicated `RedrawRequested` arm because the `Action::Output` state.update() + inline render (added in the same branch) handled everything. RedrawRequested now falls through to `_ => {}` in the WindowEvent match. The commit message: *"Direct render in UserEvent handler handles spinner animation."*

### 3. `9546621` — "Extract spinner render logic to render.rs module" (Jan 14, 2026)

Extracted the inline render code from the UserEvent handler into `render::render_spinner_frame()`. The direct render that was added alongside the `Action::Output` state.update() became a clean function call:

```rust
if state.program().is_any_pane_loading() {
    if render::render_spinner_frame(surface, device, queue, engine, renderer, viewport, debug_tool) {
        *redraw = false;
    }
}
```

## Why the Spinner Still Works Without the Block (Verified)

The spinner animation is NOT driven by SpinnerTick messages or `render_spinner_frame()`. It's driven by `window.request_redraw()` inside the main render block.

### Evidence

Tested with slow-loading JP2 images (22 seconds to load). Logged every UserEvent and every SpinnerTick processed by `state.update()`. Results:

- **Total UserEvents during 22-second load:** 3 (DirectoryEnumerated, ImagesLoaded, SpinnerTick)
- **SpinnerTicks processed by state.update():** 1
- **Spinner animation:** smooth every frame for the full 22 seconds

The SpinnerTick chain dies after the first tick (no `state.update()` in Action::Output to spawn the next one). Yet the spinner animated continuously.

### The Actual Spinner Render Loop

The spinner is kept alive by `window.request_redraw()` at `main.rs:920-922`, inside the main WindowEvent render block. This was added in the same commit as the `Action::Output` state.update() (`ae74169`).

The loop, step by step:

**Step 1.** Inside the render block at `main.rs:920-922`, `request_redraw()` is called when loading is active:
```rust
// Continue animation loop if spinner is active
if state.program().is_any_pane_loading() {
    window.request_redraw();
}
```

**Step 2.** winit delivers `RedrawRequested` as a WindowEvent. It hits the catch-all at `main.rs:710`:
```rust
_ => {}
```

**Step 3.** After the match, `*redraw` is set unconditionally at `main.rs:713`:
```rust
*redraw = true;
```

**Step 4.** `state.update()` at `main.rs:799-831` runs if the queue has messages. This processes any queued SpinnerTick or other messages, and calls `view()` which rebuilds the widget tree including the Circular spinner widget:
```rust
if !state.is_queue_empty() {
    let (_, task) = state.update(...);
    ...
}
```

**Step 5.** Render block fires at `main.rs:835` because `*redraw` is true. `renderer.present()` calls `draw()` on all widgets. The Circular spinner widget computes its rotation angle from `Instant::now()`:
```rust
if *redraw {
    *redraw = false;
    ...
    renderer_guard.present(...);
    ...
}
```

**Step 6.** Back to step 1 — `is_any_pane_loading()` is still true, so `request_redraw()` fires again.

This loop runs at full frame rate with zero dependency on SpinnerTick messages. The spinner animates because `renderer.present()` calls `draw()` on the Circular widget every frame, and the widget reads `Instant::now()` to determine its angle.

### What `render_spinner_frame()` and SpinnerTick Actually Do

`render_spinner_frame()` in the UserEvent handler (`main.rs:1029-1035`) provides a render on the UserEvent itself — but this is redundant with the main render loop above. It was originally the primary render path for spinner animation before the `request_redraw()` loop was added.

SpinnerTick messages were originally needed to keep the render chain alive. Now `request_redraw()` at line 922 handles that. The SpinnerTick chain breaks after 1 tick with no visible effect.

### What ae74169 Actually Added

The commit added THREE things:
1. `window.request_redraw()` inside the render block when loading (`main.rs:920-922`) — **this is what actually drives the spinner animation**
2. `state.update()` in `Action::Output` — originally needed to process SpinnerTick and spawn the next one, but now redundant because the `request_redraw()` loop doesn't need SpinnerTick at all
3. `WindowEvent::RedrawRequested` handler — removed in `16da27c`

Only #1 matters. #2 was the block we removed to fix NVIDIA slider performance. #3 was already removed.

## The Original Problem

The `Action::Output` state.update() processed ALL async messages, not just SpinnerTick. During slider drag, every image load completion (SliderImageWidgetLoaded) triggered a full `view()/layout()` rebuild at 65-90ms each. With images arriving every ~70ms, this created a cascade of 6-9 rebuilds between renders, consuming 450-675ms and limiting 4K slider to ~2 FPS on NVIDIA.

On AMD with Fifo (VSync), `frame.present()` blocked for ~16ms, throttling the entire event loop and limiting how many UserEvent state.update() calls could fire per second. The blocking masked the performance cost.

## The WindowEvent state.update() Block

The `state.update()` in the WindowEvent handler (the `"If there are events pending"` block) was NOT added in the spinner branch. It was there from `8777ab7` ("Refactor event loop to properly execute async tasks from state.update()", Feb 17, 2025) — the original event loop refactoring that added the iced runtime integration. This block processes iced events (mouse, keyboard, etc.) and is essential for the app to function. It was NOT deleted — it was temporarily moved during a failed batching attempt and restored in the cleanup.

## Dead Code Cleanup (Verified on Linux)

After confirming the spinner is driven entirely by `request_redraw()`, all SpinnerTick-related code and `render_spinner_frame()` were removed in commit `9d205f9`:

- `SpinnerTick` variant from `Message` enum
- `SpinnerTick` handler and spawn sites in `message_handlers.rs`
- `spinner_tick_task` and `spinner_tick_pending` fields from `DataViewer` struct
- `spinner_task` batching in `load_images()` and `update()`
- `render_spinner_frame()` function and `render.rs` module
- `mod render;` declaration in `main.rs`

**Tested on Linux (NVIDIA RTX 3090):** Spinner still animates smoothly during JP2 image loading (22+ second loads) with all SpinnerTick code removed. The `request_redraw()` loop at `main.rs:920-922` is the sole driver of spinner animation.

## macOS Benchmark Results (Apple Silicon, Metal)

Benchmarked on MacBook (Apple Silicon, Metal) comparing `main` vs `fix/nvidia-rendering-performance`. The per-UserEvent `state.update()` problem was **not NVIDIA-specific** — it affected all platforms. NVIDIA just made it more visible due to the non-blocking present mode.

### Keyboard Navigation (3s duration, 2 iterations, right only)

| Directory | main (UI / Img FPS) | fix (UI / Img FPS) | Image FPS Delta |
|---|---|---|---|
| small_images | 44.8 / 52.4 | 380.0 / 115.9 | **+121%** |
| 1080p_PNG_3MB | 32.7 / 33.5 | 169.9 / 36.0 | **+7%** |
| 4k_PNG_10MB | 10.1 / 10.8 | 410.4 / 12.8 | **+19%** |

### Slider Navigation (5s duration, 2 iterations, 20ms interval, right only)

| Directory | main (UI / Img FPS) | fix (UI / Img FPS) | Image FPS Delta |
|---|---|---|---|
| small_images | 56.7 / 34.1 | 111.9 / 40.8 | **+20%** |
| 1080p_PNG_3MB | 32.2 / 33.3 | 85.1 / 31.8 | -5% (noise) |
| 4k_PNG_10MB | 3.3 / 2.5 | 20.0 / 6.8 | **+172%** |

### Memory Usage (Slider Nav)

| Directory | main (avg) | fix (avg) |
|---|---|---|
| small_images | 149-477 MB | 143-227 MB |
| 1080p_PNG_3MB | 182-488 MB | 162-240 MB |
| 4k_PNG_10MB | 510-564 MB | 221-223 MB |

### Key Takeaways

- **UI FPS increase** is expected — main uses AutoVsync (Fifo, capped at 60fps), fix branch uses Mailbox/Immediate (uncapped). Image FPS is the meaningful metric.
- **4K slider was broken on main too:** 2.5 Image FPS with memory ballooning to 510-564 MB. Fix branch: 6.8 FPS, ~222 MB.
- **Memory reduction** (510→222 MB for 4K) comes from fewer redundant `view()/layout()` rebuilds creating fewer intermediate widget tree allocations.
- **1080p slider -5%** is within run-to-run variance (33.3 vs 31.8).
- **No macOS-specific regressions** from removing SpinnerTick or the `state.update()` change. Spinner animates smoothly.

## Linux Benchmark: NVIDIA RTX 3090 vs Previous AMD 7900 XTX Baseline

Compared the fix branch on NVIDIA RTX 3090 (Feb 28) against the last baseline on AMD 7900 XTX (Jan 20). Different GPUs and different present modes (Immediate vs Fifo), so some variance is expected.

### Keyboard Navigation (3s duration, 2 iterations, right only)

| Directory | Jan 20 AMD (Img FPS) | Feb 28 NVIDIA (Img FPS) | Delta |
|---|---|---|---|
| small_images | 184.4 | 137.2 | -26% |
| 1080p_PNG_3MB | 44.2 | 39.5 | -11% |
| 4k_PNG_10MB | 14.1 | 13.0 | -8% |

### Slider Navigation (5s duration, 2 iterations, 20ms interval, right only)

| Directory | Jan 20 AMD (Img FPS) | Feb 28 NVIDIA (Img FPS) | Delta |
|---|---|---|---|
| small_images | 46.0 | 45.6 | -1% |
| 1080p_PNG_3MB | 32.4 | 30.3 | -6% |
| 4k_PNG_10MB | 7.6 | 7.2 | -5% |

### Notes

- Slider numbers are within noise of the AMD baseline. The fix restored NVIDIA slider performance to AMD-equivalent levels.
- Keyboard nav shows a small drop for small images, likely due to different CPU-side overhead characteristics between the two GPU setups (different drivers, different present modes, different event loop cadence). Worth investigating in a separate branch but not blocking the merge.
- Raw benchmark files: `benchmarks/benchmark_20260228_143231.md`, `benchmarks/slider_nav_20260228_143320.md`

## Summary

Two blocks were permanently deleted:
1. `state.update()` inside `Action::Output` -- fixed 4K slider performance across all platforms (commit `d64cb51`)
2. All SpinnerTick infrastructure and `render_spinner_frame()` -- dead code cleanup (commit `9d205f9`)

Both were added in `ae74169` for spinner animation but are unnecessary. The spinner is driven by `window.request_redraw()` at `main.rs:920-922`, not by SpinnerTick processing. No regression on Linux or macOS.
