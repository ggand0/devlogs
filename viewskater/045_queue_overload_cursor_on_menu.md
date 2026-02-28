# Message Queue Overload & Slider Lag Investigation

**Date:** 2026-02-28
**Branch:** `fix/nvidia-rendering-performance`
**Related:** [044_nvidia_slider_fix_spinner_analysis.md](044_nvidia_slider_fix_spinner_analysis.md)

## Problem

Intermittent slider lag during drag, with `MESSAGE QUEUE OVERLOAD` warnings causing the queue to be destructively cleared. Observed on both AMD and NVIDIA, on Ubuntu. **Not consistently reproducible** -- sometimes works fine, sometimes laggy. Can go away after reboot or waiting.

---

## Finding 1: CursorOnMenu Spamming (Fixed)

```
MESSAGE QUEUE OVERLOAD: 75 pending - clearing [WindowEvent=1899, UserMsg=183, CursorMenu=4206]
```

CursorOnMenu was queued unconditionally every render frame (4206 messages). Only used in fullscreen mode.

**Fix:** Gated behind fullscreen check + only queued when value changes.

---

## Finding 2: Mouse Move Message::Event Flooding (Fixed)

After CursorOnMenu fix, mouse moves were the next source (772 per overload). `handle_event_messages()` returns `Task::none()` for mouse moves but they still trigger view()/layout() rebuild. Mouse events still reach widgets via `state.queue_event()`.

**Fix:** Filter CursorMoved/CursorLeft/CursorEntered from `Message::Event` queue.

---

## Finding 3: is_queue_empty() Triggers state.update() on Every Mouse Move

`is_queue_empty()` checks both `queued_events` AND `queued_messages`. Mouse moves queued via `queue_event()` make it return false, triggering state.update() in the WindowEvent handler on every mouse move.

```
state.update() rate: 613/s from WindowEvent, 4/s from AboutToWait
```

Each state.update() costs 6-23ms (view/layout rebuild). 613/s fully saturates the CPU.

**Fix attempted:** Gate WindowEvent state.update() on `queued_messages_len() > 0` instead of `is_queue_empty()`. Reduced to ~30/s total. Queue stays at 1, no OVERLOAD.

**Result:** Slider still laggy. state.update() frequency is not the cause.

---

## Finding 4: Rendering Is Not the Bottleneck

Full pipeline instrumentation (renders/s, avg render time, slider image load rate):

```
PIPELINE: update=23/s render=1575/s(avg=0ms,total=290ms) slider_img=8/s
```

Rendering at 1575 FPS with sub-millisecond per frame. state.update() at 23/s. But **only 8 slider images loaded per second**.

---

## Finding 5: Image I/O Is Not the Bottleneck

Per-image timing inside `create_async_image_widget_task()`:

```
SLIDER_PERF: total=0ms (read=0ms dims+handle=0ms) pos=3197 size=171KB
SLIDER_PERF: total=0ms (read=0ms dims+handle=0ms) pos=3302 size=171KB
SLIDER_PERF: total=6ms (read=6ms dims+handle=0ms) pos=3247 size=152KB
```

Images are 31-362KB and load in 0-6ms. Disk I/O is not the bottleneck.

---

## Finding 6: Stale Task Cancellation Made It Worse

Added early return in async task when `pos != LATEST_SLIDER_POS`:

```
slider_img=6/s  (down from 8/s)
```

During continuous drag, the slider position changes every state.update() call (~40ms). Tasks become stale before they execute, so most get canceled. Removing cancellation restores 8-16/s.

---

## Current Hypothesis: state.update() Round-Trip Cost

The slider image update pipeline requires two state.update() calls per image:

1. **Spawn:** state.update() processes mouse events -> slider widget generates SliderChanged -> update_pos() spawns async task (~15ms)
2. **Receive:** Async task completes in <1ms -> UserEvent queued -> next state.update() processes SliderImageWidgetLoaded (~15ms)

That's a ~30ms round-trip. Theoretical max: ~33 images/s. Observed: 16/s (rest lost to non-slider state.update() calls and rendering overhead).

The 15ms per state.update() is the fixed cost of iced's `view()` + `layout()` -- full widget tree rebuild on every call. When messages exist, iced rebuilds TWICE (source comment: "we are forced to rebuild twice for now :^)").

### Why This Hypothesis May Be Wrong

**The lag is intermittent.** The user reports it works fine sometimes and is laggy other times. The state.update() cost and round-trip pipeline are constant -- they don't explain why the slider is smooth one day and laggy the next. Something external varies:

- System load / I/O contention
- Disk cache state (hot vs cold)
- GPU driver state
- Window compositor behavior
- Memory pressure / swap
- Background processes

The investigation so far has ruled out the following as root causes:
- Message queue overflow (fixed, lag persists)
- state.update() call frequency (reduced 20x, lag persists)
- Rendering cost (sub-ms per frame)
- Image I/O cost (0-6ms per image)
- Stale task pileup (cancellation made it worse)

**The fundamental question remains: what changes between "works fine" and "laggy" runs?** The code, images, and hardware are the same.

---

## iced Internals Reference

### state.update() flow (runtime/src/program/state.rs)

1. `build_user_interface()` -- always calls `view()` + `layout()` (~15ms)
2. `user_interface.update()` -- dispatches queued events to widgets
3. If messages exist: processes messages via `program.update()`, then rebuilds widget tree AGAIN
4. `draw()` -- renders widget primitives

### is_queue_empty()

```rust
pub fn is_queue_empty(&self) -> bool {
    self.queued_events.is_empty() && self.queued_messages.is_empty()
}
```

Checks both events and messages. Mouse moves via `queue_event()` make it return false.

---

## Slider Image Update Pipeline (Full Trace)

```
Mouse move (WindowEvent)
  → queue_event(CursorMoved)                      [instant]
  → no state.update() (messages-only gate)
  → render (shows PREVIOUS image)

AboutToWait
  → state.update()                                 [~15ms: full view()+layout() rebuild]
    → processes CursorMoved events
    → slider widget emits SliderChanged
    → update_pos() checks 10ms throttle
    → spawns async task via Task::perform()
  → request_redraw()

Tokio worker thread
  → std::fs::read(path)                            [0-6ms for 30-360KB images]
  → ImageReader::into_dimensions()                  [<1ms]
  → Handle::from_bytes()                            [<1ms]
  → result sent via event_loop_proxy                [instant]

UserEvent handler
  → queue_message(SliderImageWidgetLoaded)          [instant]

Next state.update()                                 [~15ms: full view()+layout() rebuild]
  → processes SliderImageWidgetLoaded
  → sets pane.slider_image = new handle

Next render
  → finally displays the new image
```

The image loads in 0ms but requires two state.update() round-trips (spawn + receive), each costing ~15ms. state.update() runs ~25/s, yielding ~16 image updates/s while the app renders at 1575 FPS. The remaining frames re-render the same stale image.

### What's Constant vs What Varies

**Constant (doesn't explain intermittent lag):**
- state.update() cost: ~15ms per call
- Image I/O: 0-6ms
- Rendering: sub-ms per frame
- Pipeline round-trip: ~30ms

**The actual problem -- slider_img=0/s intervals:**
The pipeline data shows extended periods (up to 6 seconds) where NO images load during slider interaction, followed by normal 16/s throughput. state.update() was running 10-19/s during the zero intervals, but `create_async_image_widget_task` was never called (no SLIDER_PERF lines). This means either SliderChanged isn't reaching update_pos(), or update_pos() is returning Task::none() (throttle blocking all updates), or the Task isn't reaching the async runtime.

The 15ms state.update() cost is NOT the bottleneck -- 16 images/s is adequate for slider drag. The intermittent lag comes from the pipeline going completely idle for seconds at a time.

**Unknown (what changes between smooth and laggy runs):**
- The lag is intermittent -- same code, same images, same hardware
- Sometimes the slider is smooth, sometimes it's laggy
- Can change between app restarts or even after waiting
- Observed on both AMD and NVIDIA GPUs
- Need to instrument update_pos() to trace where the pipeline breaks

---

## Finding 7: Stale Renders Blocking state.update() (Root Cause Found)

Every CursorMoved WindowEvent was triggering:
1. `*redraw = true` (line 741, unconditional for ALL WindowEvents)
2. Full GPU render of the **same stale image** (line 897)
3. `sleep(300µs)` (line 1067)

During slider drag, hundreds of CursorMoved events arrive between AboutToWait calls. Each one renders the same stale frame + sleeps 300µs, **blocking the AboutToWait handler where state.update() actually processes slider events**.

SLIDER_TRACE instrumentation confirmed: during the 392ms gap, NO SliderChanged events were generated despite state.update() supposedly running. The event loop was stuck processing CursorMoved events (render + sleep each), never reaching AboutToWait.

**Before fix (with stale renders):**
```
PIPELINE: render=1138/s slider_img=5/s
SliderChanged gaps: 392ms, 294ms, 808ms
```

**After fix (skip render/sleep for cursor moves):**
```
PIPELINE: render=986/s slider_img=22/s
SliderChanged gaps: max ~91ms (in line with 15ms state.update() cost)
```

4x improvement in slider image throughput. The 300-400ms dead zones are eliminated.

**Fix:** Don't set `*redraw = true` and don't sleep for CursorMoved/CursorLeft/CursorEntered events. They only need `queue_event()` for widget dispatch in AboutToWait. After AboutToWait's state.update() processes them and the slider emits SliderChanged, THEN set `*redraw = true` to render the updated state.

---

## Remaining Bottleneck: state.update() Round-Trip Cost

With the stale-render fix, slider_img went from 5/s to 22/s. But this is still limited by:

1. **state.update() costs ~15ms** (full view()+layout() rebuild every call)
2. **Two round-trips per image**: spawn task (15ms) → async load (0-6ms) → receive result (15ms)
3. **10ms throttle** on Linux drops ~50% of SliderChanged events (97 throttled vs 107 processed)
4. **Theoretical max ~33/s** from the 30ms round-trip, reduced to ~22/s by throttle + overhead

The 15ms state.update() cost is iced's fixed overhead — `view()` + `layout()` rebuilds the entire widget tree. When messages exist, iced rebuilds TWICE (source: "we are forced to rebuild twice for now :^)").

This is NOT the intermittent lag — it's a constant baseline cost. The intermittent lag was caused by the stale-render blocking described in Finding 7.

---

## Applied Fixes (keep)

1. **CursorOnMenu:** Gated behind fullscreen + change detection
2. **Mouse move filtering:** CursorMoved/Left/Entered excluded from Message::Event
3. **state.update() gate:** `queued_messages_len() > 0` in WindowEvent handler (events batch to AboutToWait)
4. **AboutToWait drain:** state.update() in AboutToWait handler for event processing
5. **Skip render/sleep for cursor moves:** Don't render stale frames on CursorMoved events; render after AboutToWait state.update() instead

## Temporary Instrumentation (remove before merge)

- QUEUE_DEBUG_* atomic counters
- STATE_UPDATE_CALLS_WE/ATW counters
- RENDER_COUNT/RENDER_TIME_US counters
- SLIDER_IMG_LOADED_COUNT counter
- Pipeline rate report every 2s
- WE state.update() timing (>5ms threshold)
- AboutToWait state.update() timing (>10ms threshold)
- Event loop stall detection (last_event_time)
- SLIDER_PERF warn logs in navigation_slider.rs
- SLIDER_TRACE warn logs in navigation_slider.rs and message_handlers.rs
- Total WindowEvent timing

## Status

- Fixes 1-5: **Applied and verified**
- Stale-render root cause: **Found and fixed** (slider_img 5/s → 22/s)
- Remaining: 22/s still not enough — limited by 15ms state.update() round-trip and 10ms throttle
