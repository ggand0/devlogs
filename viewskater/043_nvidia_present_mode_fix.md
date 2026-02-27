# NVIDIA Rendering Performance Fix

**Date:** 2026-02-07
**Branch:** `fix/nvidia-rendering-performance`

## Context

After switching from AMD 7900 XTX (ROCm) to NVIDIA RTX 3090 (CUDA), the app became nearly unusable:

- Images would skate smoothly for a few seconds, then rendering would freeze
- After the freeze, old rendering events would replay without any keyboard input
- The slider UI mostly stopped working — dragging it rarely updated the displayed image
- Images only rendered once every few seconds instead of continuously

The only change was the GPU, driver (AMDGPU → NVIDIA proprietary), and compute stack (ROCm → CUDA). No code changes.

## Root Cause

**`PresentMode::AutoVsync` maps to FIFO on NVIDIA, which blocks `frame.present()` on the main thread.**

### How the cascade works

1. `ControlFlow::Poll` + skating mode → app tries to render every frame
2. `frame.present()` with FIFO on NVIDIA blocks for 10-50ms waiting for VSync when frames arrive out of sync with the display refresh cycle
3. Since `frame.present()` runs on the same thread as the event loop, **all event processing is blocked** during the stall
4. Keyboard, slider, and mouse events pile up in the winit event queue
5. The message queue grows past the 50-message threshold
6. `monitor_message_queue()` destructively clears the entire queue — including slider position events
7. Rendering stalls because no new navigation events arrive
8. When the GPU unblocks, all queued events replay at once — causing the "replay without keyboard press" effect

### Why AMD worked fine

AMD's Vulkan present queue is more forgiving with irregular frame timing. `AutoVsync` on AMD doesn't block as aggressively when frames arrive out of sync, so the event loop stays responsive even under continuous rendering load.

NVIDIA's FIFO implementation is strict — it expects frames to arrive in sync with the display refresh cycle and blocks aggressively when they don't.

## Fix

### 1. Non-blocking present mode selection (primary fix)

Query `surface.get_capabilities()` for supported present modes and select the best non-blocking option:

- **Mailbox** (preferred): Non-blocking, replaces the pending frame with the latest one. Ideal for an image viewer — always shows the most recent frame at VSync without blocking.
- **Immediate** (fallback): Non-blocking, no VSync. Used when Mailbox isn't available (e.g., some Wayland compositors, older NVIDIA drivers).
- **AutoNoVsync** (last resort): wgpu auto-selects between Mailbox and Immediate.

On the RTX 3090 with the NVIDIA proprietary driver, **Immediate** was selected (Mailbox was not reported as available).

### 2. Stored present mode in Runner state

The chosen present mode is now stored in the `Ready` state and reused during window resize (previously the resize handler hardcoded `AutoVsync` independently of the initial configuration).

## Files Changed

- `src/main.rs`:
  - Added `present_mode` field to `Runner::Ready` state
  - Added present mode capability detection in `resumed()` init block
  - Updated initial `surface.configure()` to use detected present mode
  - Updated resize handler `surface.configure()` to use stored present mode
  - Logs GPU name and available present modes at startup for diagnostics

## Cross-GPU Impact

This change is safe and beneficial for all users, not just NVIDIA. The old `AutoVsync` (Fifo) was a blocking present mode on every GPU — it just happened to not block long enough on AMD to cause visible problems. The new capability-based selection is strictly better:

| GPU | Before (AutoVsync) | After | Impact |
|-----|---------------------|-------|--------|
| AMD Linux (RADV/X11) | Fifo (blocking, worked by luck) | Mailbox (non-blocking, no tearing) | Slightly better |
| NVIDIA Linux | Fifo (blocking, broke the app) | Immediate (non-blocking) | Fixed |
| macOS (Metal) | Metal Fifo equivalent | Mailbox or Immediate | Same or better |
| Windows (DX12/Vulkan) | Fifo | Mailbox typically available | Same or better |
| Intel iGPU | Fifo | Mailbox or Immediate | Same or better |

Key architectural point: the event loop and rendering share the same thread in this app (winit single-threaded model). In apps with separate render threads, Fifo blocking is harmless — it just throttles the render loop. But here, any blocking in `frame.present()` also blocks keyboard events, slider events, async message delivery, and state updates. Even on AMD where Fifo didn't visibly break things, it was adding unnecessary VSync-wait latency during skating. The new approach eliminates that latency entirely.

## Remaining Issue: 4K Slider Performance on NVIDIA

### Problem

The present mode fix resolved keyboard navigation and event loop responsiveness, but 4K slider performance is still degraded on NVIDIA compared to AMD:

- **AMD 7900 XTX (old, AutoVsync/Fifo)**: ~6-7 FPS for 4K slider
- **NVIDIA RTX 3090 (main, AutoVsync/Fifo)**: 0 FPS (completely frozen)
- **NVIDIA RTX 3090 (this fix, Immediate)**: ~1 FPS

### Profiling Results (3840x2160 images, ~10MB each)

```
Async image load:     3-14ms per image  (fast, not bottleneck)
Renderer present:     55-58ms per frame (only when new image uploaded)
Gap between renders:  420-1465ms        (hundreds of redundant fast renders)
```

The async slider path loads 4K images in ~10ms. iced_wgpu's `renderer.present()` takes ~55ms to decode and upload a new 4K texture. At 55ms, theoretical throughput is ~18 FPS — plenty fast. But actual throughput is ~1 FPS.

### Root Cause: GPU Saturation from Uncapped Rendering

With `ControlFlow::Poll`, the event loop runs continuously. Every event sets `*redraw = true`, triggering a render. With `Immediate` present mode (non-blocking), nothing throttles the render rate. The loop spins at hundreds/thousands of FPS, rendering the same unchanged frame over and over.

These redundant renders saturate the GPU. When a new 4K slider image finally arrives and iced_wgpu needs to decode + upload it during `present()`, the GPU is too busy processing redundant render commands. The 4K texture upload gets starved.

### Why AMD Was Fine

On AMD with the old `AutoVsync` (Fifo), `frame.present()` blocked for ~16ms per frame, naturally capping the render loop at 60 FPS. Between renders, the GPU was idle and had plenty of bandwidth for 4K texture uploads. The blocking that broke keyboard navigation was ironically *helping* slider performance by acting as a frame rate limiter.

```
AMD + Fifo:      render → block 16ms → render → block 16ms → ...
                 GPU idle during blocks → texture uploads happen in gaps → 6-7 FPS

NVIDIA + Immediate: render → render → render → render → render → ...
                    GPU 100% saturated → texture uploads starved → 1 FPS
```

### Fix Attempt 1: Frame Rate Cap (MIN_FRAME_INTERVAL = 8ms)

Added a `MIN_FRAME_INTERVAL` of 8ms and a `last_render_time` timestamp check in the render block. Only render when at least 8ms have passed since the last render. This caps renders at ~125 FPS.

**Result:** No improvement. Still ~1.6-2 FPS. The frame rate cap reduced redundant renders but didn't address the actual bottleneck.

### Fix Attempt 2: SLIDER_IMAGE_DIRTY Flag

Added an `AtomicBool` flag (`SLIDER_IMAGE_DIRTY`) set when a new slider image loads via `SliderImageWidgetLoaded`. The render block checks this flag:
- During slider movement with no new image available: throttle to 30 FPS (`SLIDER_UI_FRAME_INTERVAL = 33ms`)
- When a new image is available or not sliding: use normal 8ms interval

The idea was to reduce redundant renders (cache hits) and only do heavy renders when new images actually arrive.

**Result:** No improvement. ~1.7-2 FPS. The hypothesis that redundant renders were saturating the GPU was **wrong**. The bottleneck is elsewhere.

### Profiling: Detailed Trace Logging

Added comprehensive trace logging to understand the exact timeline:
- `SLIDER_TRACE: update_pos SPAWNING` — when async image task spawns
- `SLIDER_TRACE: ImageLoaded` — when async image result arrives as UserEvent
- `BOTTLENECK: get_current_texture` — if swapchain blocks >5ms
- `BOTTLENECK: Renderer present` — when present() takes >50ms (4K decode)

**Key findings from trace data (3840x2160 PNGs):**

```
get_current_texture:    ZERO bottleneck hits (not the issue on NVIDIA)
Images loading:         Every ~70-80ms (3-4 per render cycle)
Heavy renders:          55-63ms each (PNG decode + GPU upload)
Gaps between renders:   250-570ms consistently
Heavy renders/sec:      ~2 (matching the observed 2 FPS)
Images loaded:          ~13-14 per second
Images actually shown:  ~2 per second (most overwritten before render picks them up)
```

The critical observation: images load at ~14/sec but only ~2/sec get rendered. The 250-570ms gap between heavy renders is the bottleneck.

### Fix Attempt 3: Batched Event Processing (state.update() moved within WindowEvent handler)

Moved `state.update()` from being called inline after each iced event conversion to a single call right before the render block in the WindowEvent handler. The idea was to batch 60+ mouse events into a single `view()/layout()` pass instead of 60 individual passes.

**Result:** No improvement. Gaps between heavy renders remained 250-570ms (measured via trace logs before and after). The batching was **not truly batching** because `state.update()` was still inside the WindowEvent handler, which is called once per event by winit. Moving it within the handler didn't change the call frequency.

### Root Cause: `state.update()` in UserEvent `Action::Output` handler

Added `GAP_TRACE` instrumentation to measure every `state.update()` call and every render. The evidence was immediate and conclusive:

```
Before fix (slider drag on 4K PNGs):
  UserEvent state.update():  65-90ms EACH call (full view()/layout() rebuild)
  Calls between renders:     6-9 per heavy render
  Heavy renders:             7 total during test
  UserEvent state.update()s: 50 total during test
  Total time in UE updates:  50 × ~75ms = ~3750ms (vs ~420ms in renders)
```

Between each heavy render (55-63ms), there were 6-9 UserEvent `state.update()` calls at 65-90ms each. That's 450-675ms of widget tree rebuilds between renders — exactly matching the observed 250-570ms gaps (the remainder being actual render time).

The `state.update()` call in the `Action::Output` handler was added in commit `ae74169` ("Fix spinner animation to work without mouse movement", 2026-01-03). It was introduced to process `SpinnerTick` messages immediately so the spinner could animate without mouse movement. But it processes **every** `Action::Output` message — including the high-frequency slider image load results. Each call triggers a full `view() + layout()` rebuild of the entire widget tree, which takes 65-90ms with 4K images in the tree.

### Why AMD was fine (revised understanding)

On AMD with Fifo (VSync), `frame.present()` blocks for ~16ms at VSync. This acts as a natural throttle on the entire event loop — fewer loop iterations means fewer UserEvents processed between renders, and fewer `state.update()` calls. The blocking that broke keyboard navigation was ironically limiting how often the expensive UserEvent `state.update()` could fire.

On NVIDIA with Immediate, `frame.present()` returns instantly. The event loop runs at full speed, processing UserEvents as fast as they arrive. Every async image load result immediately triggers a 75ms `state.update()`, creating a cascade that dominates the render cycle.

### Fix: Remove `state.update()` from UserEvent `Action::Output`

The fix is a single change: remove the immediate `state.update()` call from the `Action::Output` handler. Messages are still queued via `state.queue_message()` and processed by the existing `state.update()` in the WindowEvent handler on the next event.

**After fix:**
```
state.update() calls between renders: 1 (in WindowEvent handler, ~70ms)
Heavy render:                         ~58ms
Total cycle:                          ~130ms → ~7.5 FPS
```

Performance restored to match AMD levels (~6-7 FPS for 4K slider).

### Why the spinner still works

The `state.update()` was originally added for spinner animation. The spinner still works because:

1. `render_spinner_frame()` was added later as a separate rendering path in the UserEvent handler — it doesn't depend on `state.update()` being called immediately
2. The spinner widget computes animation from wall clock time (commit `ae74169`), not from message processing
3. Queued messages are still processed on the next WindowEvent's `state.update()` — the SpinnerTick chain continues with at most a one-frame delay
4. During loading (when spinners are active), `window.request_redraw()` keeps the event loop pumping WindowEvents

### Why the earlier "batching" attempts failed

Fix Attempt 3 moved `state.update()` within the WindowEvent handler, but the handler is called once per event by winit. Moving code within the handler doesn't change call frequency. The real problem was the **UserEvent** `state.update()`, not the WindowEvent one.

The WindowEvent `state.update()` was already relatively cheap (processing 1-2 queued items per call). The UserEvent `state.update()` was the killer because it rebuilt the entire widget tree (including 4K image widgets) every time an async message arrived.

## Notes

- The `monitor_message_queue()` destructive clearing at threshold 50 is a separate concern — it's a safety valve but can drop important events like slider positions. The present mode fix eliminates the condition that triggers it (event loop stalling → queue overflow).
- `desired_maximum_frame_latency` was tested at both 1 and 2 — made no difference to 4K slider FPS. Kept at 2 (original value).
- The `state.update()` still exists in the WindowEvent handler (called once per event). This could be further optimized by moving it to the `AboutToWait` handler for true per-frame batching, but it's not necessary for the current fix since WindowEvent `state.update()` calls are much cheaper than the UserEvent ones were.
