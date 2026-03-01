# 048: Drag Budget Profiling

## Context

After fixing the render feedback loop (047), slider drag went from ~20 FPS to 63-85 FPS render rate. But image update rate is only 26-28/s, and there are unexplained gaps and render time spikes. This devlog documents the per-phase profiling results.

## Instrumentation

Split the render block into separate measurements:

- **`renderer.present()`** — iced_wgpu CPU-side: widget tree traversal, draw command prep, texture atlas ops, GPU command encoding
- **`engine.submit()`** — wgpu command buffer submission to GPU
- **`gap`** — time between render end (`LAST_RENDER_END`) and next render start (the `if *redraw` block)
- **`state.update()`** — iced's view/diff/layout/draw rebuild in AboutToWait handler
- **`frame.present()`** — swapchain frame presentation (Immediate mode)
- **`surface.get_current_texture()`** — swapchain buffer acquisition

## Results (NVIDIA RTX 3090, Immediate present mode)

### Per-second averages during drag

| Phase | Period 1 (63fps) | Period 2 (85fps) |
|-------|-------------------|-------------------|
| renderer.present() | **8.6ms** | **5.3ms** |
| engine.submit() | 0.1ms | 0.1ms |
| gap between renders | **6.7ms** | **5.5ms** |
| state.update() (ATW) | 0.3ms | 0.2ms |
| frame.present() | 0ms | 0ms |
| surface.get_current_texture() | 0ms | 0ms |
| **Total per frame** | **15.4ms** | **10.9ms** |

### Per-second pipeline stats

| Metric | Period 1 | Period 2 |
|--------|----------|----------|
| render/s | 63 | 85 |
| state.update/s (ATW) | 63 | 85 |
| state.update/s (WE) | 17 | 16 |
| slider_img/s | 28 | 26 |
| WE redraws during drag | 0 | 0 |
| UE redraws during drag | 29 | 27 |

### Per-frame render time (individual samples)

```
31, 10, 29, 14, 6, 14, 28, 15, 22, 27, 29, 15, 7, 22,
26, 18, 15, 15, 29, 11, 13, 30, 14, 30, 10, 15, 16, 29,
5, 17, 23, 28, 13, 13, 22, 11, 11, 13, 12, 29, 18, 17,
10, 16, 11, 21, 20, 24, 18, 24, 15, 18, 6, 24, 12, 26,
17, 12, 9, 10, 26, 20, 10, 13, 10, 12, 14, 14, 6, 11,
15, 13, 10, 31, 11
```

Range: 5-31ms. Bimodal distribution — clusters around 10-15ms and 25-31ms.

## Findings

### 1. engine.submit() is NOT the bottleneck

At 0.1ms per call, wgpu command submission is essentially free. The GPU is not stalling on command buffer submission or fence synchronization.

### 2. renderer.present() is the dominant GPU cost

5-9ms average, but with spikes to 29-31ms. This is iced_wgpu's CPU-side rendering pipeline:
- Widget tree traversal to generate draw primitives
- Image atlas operations (add/remove textures)
- GPU command encoding into the command buffer

The bimodal timing pattern (alternating ~10ms and ~30ms frames) suggests the 30ms frames coincide with new image texture uploads to the GPU atlas. With images arriving at ~28/s and renders at ~75/s, roughly 1 in 3 renders processes a new image.

### 3. Unexplained 5.5-6.7ms gap between renders

The gap measures time from one render's end to the next render's start. Known costs within this gap:
- `state.update()`: 0.2-0.3ms
- `sleep(300µs)` on RedrawRequested: 0.3ms
- Cursor event processing: ~0.1ms per event

Total accounted: ~0.7ms. **Remaining ~5ms is unaccounted.**

Possible sources:
- winit event dispatch overhead between iterations
- Mutex lock contention (FRAME_TIMES, LAST_RENDER_TIME, LAST_RENDER_END, CURRENT_FPS — all acquired in the render path)
- OS/compositor scheduling latency
- Time between `window.request_redraw()` in ATW and delivery of `RedrawRequested` in the next WE batch

### 4. Image update rate limited to 26-28/s

Separate from the render rate (63-85fps). The image rate is limited by:
- 10ms throttle in `navigation_slider::update_pos()` on Linux (`PLATFORM_THROTTLE_MS`)
- Async image load round-trip: SliderChanged → update_pos → Task::perform → decode → SliderImageWidgetLoaded → next state.update() → view rebuild → render
- Each image requires a full message pipeline round-trip across multiple event loop iterations

## Open Questions

1. **What causes the bimodal renderer.present() spikes?** Is it atlas texture upload when a new image arrives? Need to instrument inside iced_wgpu's `renderer.present()` to confirm.

2. **Where is the 5ms gap?** Need to add timestamps at more points between render end and next render start (post-render cleanup, WE event processing, ATW entry, ATW exit, next render entry).

3. **Can the image update rate exceed 28/s?** The 10ms throttle caps at 100/s, but the actual rate is 28/s due to the async round-trip. Reducing the throttle alone won't help — the pipeline latency is the bottleneck.

4. **Does this match AMD performance?** Need equivalent profiling on AMD GPU to see if the bimodal pattern and 5ms gap are NVIDIA-specific.

## Changes Made (not committed)

- Split `renderer.present()` and `engine.submit()` timing in render block
- Added `LAST_RENDER_END` tracking for inter-frame gap measurement
- Added `DRAG_BUDGET` per-second summary log with present/submit/gap/atw_update breakdown
- Added ATW state.update() accumulator during drag
- Removed ATW render gate (was 8ms, now unconditional `*redraw = true` when queue is not empty)
