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

## Why the Spinner Still Works Without the Block

After `16da27c` removed the RedrawRequested handler and `9546621` extracted `render_spinner_frame()`, the spinner loop changed to:

```
1. SpinnerTick arrives as UserEvent → Action::Output
2. state.queue_message(SpinnerTick) — queued, NOT processed
3. *redraw = true
4. render_spinner_frame() fires — renders spinner using Instant::now() (wall clock)
5. AboutToWait: window.request_redraw() because *redraw is true
6. RedrawRequested falls to _ => {} in WindowEvent match
7. After match: *redraw = true
8. state.is_queue_empty() is FALSE (SpinnerTick still queued from step 2)
9. state.update() processes SpinnerTick → returns Task for next tick → runtime.run()
10. Render block fires (because *redraw is true)
11. Next SpinnerTick arrives → back to step 1
```

Two things make this work:

1. **`render_spinner_frame()` uses wall clock time** — the Circular widget computes its rotation from `Instant::now()`, not from message state. It doesn't need SpinnerTick to be processed before rendering. SpinnerTick just triggers the scheduling of the next tick.

2. **The message chain continues through the WindowEvent path** — `window.request_redraw()` generates a `RedrawRequested` WindowEvent. Even though it falls to `_ => {}`, the code after the match runs `state.update()` which processes the queued SpinnerTick and spawns the next one via the runtime.

## The Original Problem

The `Action::Output` state.update() processed ALL async messages, not just SpinnerTick. During slider drag, every image load completion (SliderImageWidgetLoaded) triggered a full `view()/layout()` rebuild at 65-90ms each. With images arriving every ~70ms, this created a cascade of 6-9 rebuilds between renders, consuming 450-675ms and limiting 4K slider to ~2 FPS on NVIDIA.

On AMD with Fifo (VSync), `frame.present()` blocked for ~16ms, throttling the entire event loop and limiting how many UserEvent state.update() calls could fire per second. The blocking masked the performance cost.

## The WindowEvent state.update() Block

The `state.update()` in the WindowEvent handler (the `"If there are events pending"` block) was NOT added in the spinner branch. It was there from `8777ab7` ("Refactor event loop to properly execute async tasks from state.update()", Feb 17, 2025) — the original event loop refactoring that added the iced runtime integration. This block processes iced events (mouse, keyboard, etc.) and is essential for the app to function. It was NOT deleted — it was temporarily moved during a failed batching attempt and restored in the cleanup.

## Known Regression: SpinnerTick Chain Breaks When Idle

`render_spinner_frame()` is called in the **UserEvent handler** (line 1030), right after `Action::Output` queues the message. It renders and sets `*redraw = false`. This means:

1. AboutToWait sees `*redraw = false` → no `window.request_redraw()`
2. No RedrawRequested WindowEvent → no WindowEvent handler → no `state.update()`
3. The queued SpinnerTick is never processed → no Task spawned for next tick
4. The tick chain dies

The spinner appears to work during testing because mouse movement generates WindowEvents that trigger `state.update()` and process the queued ticks. But if the user stops moving the mouse while an image loads, the spinner will freeze.

Fix options:
- Call `window.request_redraw()` after `render_spinner_frame()` when loading is active (simplest)
- Add a targeted `state.update()` only when `is_any_pane_loading()` is true in the UserEvent handler
- Move the SpinnerTick chain to a timer-based mechanism that doesn't depend on message processing

## Summary of What Was Deleted

Only ONE block was permanently deleted: the `state.update()` inside `Action::Output`. This block was added in `ae74169` for spinner animation but became unnecessary for rendering after `render_spinner_frame()` was extracted as a separate path using wall clock time. However, it was still needed to keep the SpinnerTick task chain alive — this is now a regression that needs to be addressed.
