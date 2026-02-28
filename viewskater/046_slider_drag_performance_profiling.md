# Slider Drag Performance Profiling

**Date:** 2026-02-28
**Branch:** `fix/nvidia-rendering-performance`
**Related:** [045_queue_overload_cursor_on_menu.md](045_queue_overload_cursor_on_menu.md), [043_nvidia_present_mode_fix.md](043_nvidia_present_mode_fix.md)

## Setup

Switched Cargo.toml from remote git iced dependencies to local path (`../iced/`) to instrument the iced runtime directly. Added microsecond-precision timing to every phase of the iced pipeline.

Instrumented files:
- `iced/runtime/src/program/state.rs` — `state.update()` phases
- `iced/runtime/src/user_interface.rs` — `UserInterface::build()` (diff vs layout)
- `src/main.rs` — render breakdown (`get_current_texture`, `renderer.present`, `frame.present`), event gap tracking

GPU: NVIDIA GeForce RTX 3090
Present mode: Immediate (non-blocking)

---

## Finding 1: state.update() Phase Breakdown

| Phase | Typical Range | % of Total |
|-------|--------------|------------|
| `view()` | 100–212 µs | ~1% |
| `diff()` | 9–39 µs | <0.5% |
| `layout()` | 4–26 ms | **97–99%** |
| event processing | 14–81 µs | <0.5% |
| `draw()` | 43–99 µs | <0.5% |
| **Total** | **4–26 ms** | |

Stats (n=44 samples during drag):
- min: 3.8 ms, max: 26.5 ms, avg: 13 ms, median: 10 ms, p90: 20 ms, p95: 22 ms

**layout() dominates state.update() completely.** view(), diff(), event dispatch, and draw() are all sub-millisecond and negligible.

---

## Finding 2: Render Breakdown — frame.present() Stalls

| Phase | Typical | Worst |
|-------|---------|-------|
| `surface.get_current_texture()` | 5–600 µs | 1.7 ms |
| `renderer.present() + engine.submit()` | 70–400 µs | 9.5 ms |
| **`frame.present()`** | **65–1000 µs** | **42 ms** |

frame.present() stats (n=140 renders logged):
- Average: 1019 µs
- \>1 ms: 32 renders (22%)
- \>5 ms: 4 renders (3%)
- \>10 ms: 2 renders (1%) — **18 ms and 42 ms stalls observed**

Despite using `Immediate` present mode, the NVIDIA driver still blocks on `frame.present()` for up to 42 ms. During this time the entire event loop is frozen — no cursor events, no state updates, nothing.

Confirmed by EVENT_GAP correlation:
```
T=26.448  RENDER: frame_present=41306us total=42348us
T=26.449  EVENT_GAP: 43ms since last event
```

---

## Finding 3: Redundant Render Loop (Spinner)

Pipeline stats (2s window):
- Updates: 84/2s = 42/s
- Renders: 735/2s = 367/s
- **Ratio: ~9 renders per state update**

The spinner animation code triggers continuous redraws:
```rust
// src/main.rs line ~1009
if state.program().is_any_pane_loading() {
    window.request_redraw();
}
```

During slider drag, `loading_started_at` may be set (from the initial SliderChanged message), so `is_any_pane_loading()` returns true. This creates a tight render loop that fires ~9x more renders than actual state updates.

Each redundant render is a chance to hit a frame.present() GPU stall, multiplying the stall frequency by ~9x.

---

## Time Budget Analysis (2-second window)

| Activity | Time | % |
|----------|------|---|
| state.update() | ~540 ms | 27% |
| Renders (total) | ~246 ms | 12% |
| **Unaccounted (event loop idle / GPU stalls / scheduling)** | **~1214 ms** | **61%** |

The majority of wall-clock time is NOT spent in state.update() or rendering. It's lost to:
1. GPU stalls in frame.present() blocking the event loop
2. Redundant render cycles from the spinner loop
3. OS/winit event delivery latency between iterations

---

## Drag Poll Interval Stats

Time between consecutive DRAG_DIRECT polls (n=42):
- min: 13 ms, max: 150 ms, avg: 46 ms, median: 35 ms, p90: 80 ms

state.update() accounts for ~10 ms median. The remaining 25 ms median gap is render overhead + frame.present() stalls + event delivery.

---

## Root Causes

1. **frame.present() GPU stalls (up to 42 ms)** — NVIDIA Immediate present mode does not prevent blocking. Each stall freezes the entire event loop.
2. **Spinner render loop (9x redundant renders)** — Amplifies the probability of hitting a GPU stall by rendering ~367 times/second when only ~42 updates/second occur.
3. **layout() cost (4–26 ms)** — The dominant cost within state.update(), though state.update() itself is only part of the total loop time.

## Finding 4: NVIDIA Driver — frame.present() Blocking is a Known Issue

**Driver version:** 590.48.01 (post-560 fix)
**Present mode:** Immediate

Research confirms this is a known NVIDIA Vulkan driver behavior:

- **Immediate mode reduces but does not eliminate stalls.** NVIDIA forum: "it seems immediate happens too, just less." ([NVIDIA Forum: VkQueuePresent takes too long](https://forums.developer.nvidia.com/t/vkqueuepresent-takes-too-long-blocking-the-frames/265403))
- **The 560 beta driver fixed a Wayland-specific blocking bug** where `vkQueuePresentKHR` synchronously waited for GPU completion. Our driver (590) includes this fix. ([NVIDIA Forum: Vulkan/Wayland vkQueuePresentKHR](https://forums.developer.nvidia.com/t/vulkan-wayland-vkqueuepresentkhr-waits-for-gpu-to-finish/302203))
- **On X11, blocking still happens at driver level.** The NVIDIA Vulkan team confirmed it as a bug (Sep 2023) with "a fix in flight," but X11 fix status is unclear.
- **wgpu has no async present API.** `SurfaceTexture::present()` is synchronous by design. [wgpu issue #2650](https://github.com/gfx-rs/wgpu/issues/2650) was closed as "not planned." No application-side workaround exists.
- **The real multiplier is redundant renders.** The driver stall itself is unfixable from our side, but rendering 367/s instead of 42/s means we hit the stall ~9x more often than necessary.

---

## Potential Fixes

1. **Suppress spinner render loop during drag** — Stop `window.request_redraw()` from `is_any_pane_loading()` while slider is actively being dragged. This eliminates ~9x redundant renders and proportionally reduces exposure to frame.present() GPU stalls. Highest impact, simplest change.
2. **Throttle renders to match update rate** — Only render after state.update() produces new content. No point rendering the same frame 9 times.
3. **Skip layout during drag** — Cache `layout::Node` and reuse it when bounds haven't changed. Reduces state.update() from 10 ms median to ~1 ms (since layout is 97-99% of the cost). Second-highest impact.
4. **Decouple presentation to a separate thread** — Move `frame.present()` off the main event loop so GPU stalls don't block cursor events. Complex but would fully eliminate the stall impact. wgpu's synchronous API makes this difficult.
