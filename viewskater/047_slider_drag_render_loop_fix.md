# 047: Slider Drag Render Loop Fix

## Problem

During slider drag, the app ran at ~16-20 FPS despite `state.update()` taking 0ms (after the `known_dimensions` fix). The slider handle lagged behind the cursor and images updated slowly.

## Root Cause: Render Feedback Loop

The event loop had a feedback loop that submitted 1000+ GPU frames per second during drag:

1. **AboutToWait** handler unconditionally set `*redraw = true` and called `window.request_redraw()`
2. winit delivered `WindowEvent::RedrawRequested` on the next iteration
3. **WindowEvent** handler set `*redraw = true` for all non-cursor events (line 760) and also for all non-cursor iced events (line 783)
4. Render block executed (`if *redraw`) — submitted frame to GPU
5. Back to step 1

This produced `render=1132/s` but only `slider_img=16/s`. **98.6% of renders showed the exact same stale image.**

### Why 1132 renders caused 16 FPS

Each render blocks the event loop during `renderer.present() + engine.submit()` (iced_wgpu GPU pipeline). With 1132 renders/s, the event loop was blocked most of the time. CursorMoved events and async image load results could only be processed in tiny gaps between renders, limiting image throughput to ~16/s.

### Why the initial 8ms render gate failed

The first attempted fix: a time-based gate at the render site checking `LAST_RENDER_TIME.elapsed() < 8ms`. This failed because `LAST_RENDER_TIME` was set at render **start**. After a render that took 13ms+ of GPU time, `elapsed = 13ms > 8ms`, so the gate immediately passed for the next render. The gate had zero effect.

## Fix: Break the Feedback Loop (3 changes)

### 1. WindowEvent handler: suppress redraws during drag

Both `*redraw = true` sites in the WE handler (line 760 for raw WindowEvents, line 783 for iced events) were guarded with `!state.program().is_slider_moving`. During drag, no WindowEvent can trigger a render — only the ATW handler controls render timing.

```rust
// Line 760: raw WindowEvent redraw
if !is_cursor_move_event && !state.program().is_slider_moving {
    *redraw = true;
}

// Line 783: iced event redraw
if !is_cursor_move {
    state.queue_message(Message::Event(event.clone()));
    if !state.program().is_slider_moving {
        *redraw = true;
    }
}
```

### 2. UserEvent handler: suppress redraws during drag

The UserEvent handler unconditionally set `*redraw = true` for all actions (Action::Output, Action::Widget, etc.). During drag, the queued message is still processed in ATW's `state.update()` — we just don't trigger an immediate render.

```rust
if !state.program().is_slider_moving {
    *redraw = true;
}
```

### 3. AboutToWait handler: gate redraws at ~60fps

Instead of unconditionally setting `*redraw = true`, the ATW handler checks `LAST_RENDER_TIME` (now set at render **end**) and only requests a render every 16ms during drag.

```rust
if state.program().is_slider_moving {
    if let Ok(last) = LAST_RENDER_TIME.lock() {
        if last.elapsed().as_millis() >= 16 {
            *redraw = true;
        }
    }
} else {
    *redraw = true;
}
```

### 4. LAST_RENDER_TIME moved to render end

Previously set at render start (before GPU work). Now set after `frame.present()` completes. This ensures the 16ms gate measures from render completion, giving the event loop guaranteed processing time between renders.

## Results

| Metric | Before | After |
|--------|--------|-------|
| render/s | 1132 | 30 |
| slider_img/s | 16 | 32 |
| frame_fps | ~20 | 31 |
| WE redraws during drag | 1178 | 0 |
| state.update() | 0ms | 0ms |
| avg render time | <1ms (misleading*) | 13ms |

*Integer division artifact: 1132 renders averaging 0.5ms each rounds to 0ms in the stats.

## Remaining Bottleneck: renderer.present() GPU time

Per-phase render breakdown during drag:
```
DRAG render: 20ms (acquire=0ms gpu=20ms present=0ms)
DRAG render: 39ms (acquire=0ms gpu=39ms present=0ms)
DRAG render: 16ms (acquire=0ms gpu=16ms present=0ms)
```

- `acquire` = `surface.get_current_texture()` — 0ms, swapchain is not the issue
- `gpu` = `renderer.present() + engine.submit()` — 9-39ms, **this is the bottleneck**
- `present` = `frame.present()` — 0ms, frame presentation is instant with Immediate mode

The `gpu` phase is iced_wgpu's rendering pipeline: traversing the widget tree, preparing draw commands, uploading new image textures to the GPU atlas, and submitting GPU commands. Average 13ms per render on NVIDIA RTX 3090.

At 13ms per render, max practical FPS ≈ 1000/13 ≈ 76fps. But with 16ms gate + 13ms render = 29ms cycle ≈ 34fps. The 31fps observed matches this prediction.

### Possible next steps for GPU time reduction

- Split `renderer.present()` and `engine.submit()` timing to isolate CPU vs GPU work
- Investigate whether iced_wgpu atlas operations (add/remove textures during rapid image changes) are expensive
- Check if reusing the same `image::Handle` ID across slider loads avoids atlas reallocation
- Consider pre-uploading textures on a separate thread

## Other Changes in This Session

### known_dimensions fix (viewer.rs)

Added `known_dimensions: Option<(u32, u32)>` to the Viewer widget. When set, `layout()` skips `renderer.measure_image()` which was triggering full image decode from disk (4-26ms for each layout pass). Now layout is 0ms during drag.

Wired up in `pane.rs` and `ui.rs` to pass `slider_image_dimensions` to the Viewer.

### Spinner suppression during drag

The spinner animation loop (`window.request_redraw()` after each render when `is_any_pane_loading()`) is suppressed during slider drag to avoid multiplying render submissions.
