# 049: render_spinner_frame Was Not Dead Code

## Context

Commit `9d205f9` ("Remove SpinnerTick dead code and render_spinner_frame") removed SpinnerTick messages and `render_spinner_frame()` from the UserEvent handler, based on analysis showing the spinner animation was driven by `window.request_redraw()` in the render block, not by SpinnerTick messages.

Slider drag stopped working after this commit. Testing confirmed: `4d78ea1` (before removal) has working slider drag, `9d205f9` (after removal) does not.

## Root Cause: CursorOnMenu Feedback Loop

`render_spinner_frame()` was accidentally preventing a render feedback loop during slider drag.

### Why is_any_pane_loading() is true during drag

`loading_started_at` is set during slider navigation (`navigation_slider.rs:281`), not just during initial directory loading. So `is_any_pane_loading()` returns true during slider drag.

### The feedback loop (exposed at 9d205f9)

The render block unconditionally queues a `CursorOnMenu` message every render:

```rust
// In the render block (after renderer.present())
state.queue_message(Message::CursorOnMenu(
    !state.program().cursor_on_footer
    && state.mouse_interaction() == mouse::Interaction::Pointer));
```

This creates a feedback loop:
1. Any event → `*redraw = true` → render block fires
2. Render block queues `CursorOnMenu` message
3. Next WE iteration: `!state.is_queue_empty()` → state.update() → view()/layout() rebuild
4. state.update() processes `CursorOnMenu` → triggers another rebuild
5. `*redraw = true` → render block fires again → back to step 2

With cursor events arriving at 100-200Hz during drag, this multiplies into 1000+ renders/s of stale frames, starving actual image updates.

### How render_spinner_frame masked it (at 4d78ea1)

In the UserEvent handler at 4d78ea1:

```rust
*redraw = true;

// This code ran during slider drag because is_any_pane_loading() was true
if state.program().is_any_pane_loading() {
    if render::render_spinner_frame(
        surface, device, queue, engine, renderer, viewport, debug_tool,
    ) {
        *redraw = false;  // <-- Prevents normal render block from running
    }
}
```

`render_spinner_frame()` did a lean render (just present + submit, no bookkeeping) and set `*redraw = false`. This **skipped the normal render block** for UserEvents during drag, which meant:
- No `CursorOnMenu` message queued for those renders
- Fewer state.update() calls triggered
- The feedback loop didn't amplify

Without `render_spinner_frame` (9d205f9), every UserEvent goes through the normal render path → CursorOnMenu queued → feedback loop runs at full speed.

## The Real Fix

The unconditional `CursorOnMenu` message is the root cause. It should only be queued when the value actually changes, and only in fullscreen mode (it's only used for showing/hiding the menu bar on hover).

```rust
// Fix: only queue when value changes, and only in fullscreen
if state.program().window_state == WindowState::FullScreen {
    let new_val = !state.program().cursor_on_footer
        && state.mouse_interaction() == mouse::Interaction::Pointer;
    if new_val != state.program().cursor_on_menu {
        state.queue_message(Message::CursorOnMenu(new_val));
    }
}
```

This eliminates the feedback loop entirely without needing `render_spinner_frame()` as a band-aid.

## Fix Applied

Replaced the unconditional `CursorOnMenu` with a conditional version (fullscreen only + value changed). This eliminates a major source of redundant renders during drag.

The slider lag is intermittent — it appears and disappears across runs with no code changes. Root cause not yet determined.

During debugging, `git checkout 4d78ea1` was tested and slider drag worked there. This initially pointed to `9d205f9` (render_spinner_frame removal) as the cause. However, after applying the CursorOnMenu fix and then stashing it, `9d205f9` also worked without any changes — confirming the lag is intermittent, not deterministically caused by the render_spinner_frame removal.

## Benchmark Results (Jan 20 vs Mar 1)

GPU: NVIDIA RTX 3090, Present mode: Immediate

### Keyboard Navigation (3s duration, 2 iterations)

| Directory | Jan 20 Image FPS | Mar 1 Image FPS | Delta |
|-----------|-----------------|-----------------|-------|
| small_images | 184.4 | 202.7 | +10% |
| 1080p_PNG_3MB | 44.2 | 43.0 | -3% |
| 4k_PNG_10MB | 14.1 | 14.0 | -1% |

### Slider Navigation (5s duration, 2 iterations, 20ms interval)

| Directory | Jan 20 Image FPS | Mar 1 Image FPS | Delta |
|-----------|-----------------|-----------------|-------|
| small_images | 46.0 | 44.3 | -4% |
| 1080p_PNG_3MB | 32.4 | 30.4 | -6% |
| 4k_PNG_10MB | 7.6 | 7.2 | -5% |

### Memory (Slider Nav)

| Directory | Jan 20 | Mar 1 |
|-----------|--------|-------|
| small_images | 154-412 MB | 227-428 MB |
| 1080p_PNG_3MB | 265-412 MB | 292-428 MB |
| 4k_PNG_10MB | 411-413 MB | 427-475 MB |

No significant regression. Slider image FPS is 4-6% lower across the board, within run-to-run variance given the intermittent nature of the lag. The benchmark replay path may not trigger the same feedback loop that manual drag does.

## Archived Commits

Two experimental commits were moved to `archive/nvidia-fix-experiments`:
- `827bf81` — Skip rendering on cursor-move events to reduce stale frame overhead
- `859a688` — Add drag render budget instrumentation and suppress redraw feedback loop

The CursorOnMenu fix was attempted as part of 827bf81 but didn't work at the time. Additional changes (cursor move filtering, WE/UE redraw suppression, ATW state.update, instrumentation) were added on top trying to fix it, but those changes broke slider drag independently. The CursorOnMenu fix was extracted and applied cleanly to 9d205f9.

## Loading Spinner Fix

The CursorOnMenu fix had a side effect: it broke the loading spinner animation.

### Why

The spinner widget computes its rotation angle from `Instant::now()` inside `draw()`. For the animation to advance, `state.update()` must run each frame — it calls `view()` to rebuild the widget tree and `draw()` to produce updated render output.

Before the CursorOnMenu fix, the unconditional `CursorOnMenu` message kept `state.is_queue_empty()` false every frame, so `state.update()` always ran during loading. After the fix removed that message, the queue was empty during loading and `state.update()` was skipped — the spinner froze.

### Fix

Widened the `state.update()` guard in main.rs from:

```rust
if !state.is_queue_empty() {
```

to:

```rust
if !state.is_queue_empty() || state.program().is_any_pane_loading() {
```

This ensures `state.update()` runs every frame while loading is in progress (spinner animates), and stops once loading completes (`is_any_pane_loading()` returns false, no extra cost at steady state).
