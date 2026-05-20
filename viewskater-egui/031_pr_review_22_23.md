# 031 -- PR #22 and #23 review (YelovSK)

Both PRs are from YelovSK (Patrik Hampel), a first-time contributor. Self-described
as having zero Rust experience, using AI for syntax. Both touch only `src/pane.rs`.

## PR #23 -- Use smooth scroll and align wheel zoom input

### The bug

Scroll zoom without Ctrl (mouse_wheel_zoom enabled) feels choppy compared to
Ctrl+scroll zoom. Both go through `zoom_image()` but read different inputs:

- **Ctrl+scroll**: egui consumes the scroll events and produces `zoom_delta()`,
  which internally reads `smooth_scroll_delta`. egui applies exponential
  smoothing over multiple frames before returning the value.
- **Scroll zoom without Ctrl**: our code reads `raw_scroll_delta.y`, which gives
  discrete wheel ticks with no smoothing. Each notch is a single jump.

The smoothness difference has nothing to do with Ctrl itself. It's just that
`zoom_delta()` happens to use the smoothed input, while our scroll path was
reading the raw input.

### Initial version (commit 020a5c4)

1. Switches `raw_scroll_delta` to `smooth_scroll_delta` so scroll zoom without
   Ctrl gets the same frame-by-frame interpolation.
2. Removes the `if scroll != 0.0` guard, since `(0.0 * speed).exp() = 1.0`.
3. Hardcodes `scroll_zoom_speed = 0.003` and rescales `zoom_delta()` (pinch)
   via `pinch.powf(0.003 / egui_zoom_speed)` to match.

### Review concern: speed choice

The contributor hardcoded 0.003 (carried over from the original code) and wrote
rescaling math to bring Ctrl+scroll down from egui's default 0.005 to match.
Both paths felt slow at 0.003 on testing. We asked to bump to 0.005, which
matches egui's default (`1.0 / 200.0` in
[`memory/mod.rs:335`](https://github.com/emilk/egui/blob/0.31.1/crates/egui/src/memory/mod.rs#L335)).

### Updated version (commit 7387cd0)

The contributor took a cleaner approach than just changing the constant:

1. Defines `SCROLL_ZOOM_SPEED = 1.0 / 200.0` as a constant in `app.rs`.
2. Sets it on egui's options at startup: `cc.egui_ctx.options_mut(|o| o.scroll_zoom_speed = SCROLL_ZOOM_SPEED)`.
   This means both egui's internal Ctrl+scroll path and our scroll path read
   from the same source.
3. In `pane.rs`, reads the speed from `ui.ctx().options(|o| o.scroll_zoom_speed)`
   instead of hardcoding.
4. Drops the `pinch_factor` rescaling entirely. Since both paths now use the
   same option value, `zoom_delta()` and our scroll factor are guaranteed to
   be in sync. No rescaling needed.
5. Still switches `raw_scroll_delta` to `smooth_scroll_delta` and removes the
   zero-check guard.

This is simpler and more correct. Changing the speed in the future only requires
editing the constant in one place.

### egui internals (for reference)

`smooth_scroll_delta` is documented on `InputState` in egui:
https://docs.rs/egui/latest/egui/struct.InputState.html

The smoothing logic lives in `egui/src/input_state/mod.rs` (egui 0.31.1). It
uses `exponential_smooth_factor(0.90, 0.1, dt)` to spread large discrete scroll
ticks across multiple frames. Small deltas (< 1.0) pass through unsmoothed.

When a zoom modifier (Ctrl/Cmd) is held, egui routes scroll events into a
separate `smooth_scroll_delta_for_zoom` accumulator and bakes it into the
`zoom_factor_delta` returned by `zoom_delta()`. The `scroll_zoom_speed` option
(default `1.0 / 200.0` = 0.005) controls the scaling.

Related egui issues:
- https://github.com/emilk/egui/issues/4777 (Ctrl+scroll zoom regression)
- https://github.com/emilk/egui/issues/7650 (scroll to zoom without Ctrl in Scene)

### Status

Updated version reviewed. Ready to merge.

---

## PR #22 -- Toggle actual size on double-click (+18 / -4)

### What it does

Double-click toggles between fit-to-screen and actual size.
Standard image viewer behavior (macOS Preview, IrfanView, XnView, etc.).

Currently double-click resets zoom and pan unconditionally. The PR replaces that
with a toggle:

- **At fit (zoom == 1.0)**: zoom to actual size (`actual_size_zoom = 1.0 / scale`).
  `scale` is `min(window_width / image_width, window_height / image_height)`,
  the factor used to fit the image in the window. Dividing by it reverses the
  shrink to show real pixels.
- **At any other zoom level**: reset to fit (zoom = 1.0, pan = zero). Same as
  old behavior.

### Zoom-toward-cursor pan

When zooming into actual size, the code pans so the spot under the cursor stays in
place:

```rust
self.pan = (hover_pos - available.center()) * (1.0 - self.zoom);
```

This matches the formula in `zoom_image()` (line 412) when starting from
zoom=1.0 and pan=zero. Verified correct.

### Small image case

If the image is smaller than the window, `actual_size_zoom < 1.0` (the image
gets smaller to show at real size). In this case the code skips cursor-tracking
and just centers the image (`pan = zero`). If the image perfectly fits the
window, `actual_size_zoom = 1.0` and double-click does nothing, which is
correct since fit and actual size are the same.

### The `is_fit_to_screen` check

Uses `(self.zoom - 1.0).abs() < f32::EPSILON`, so only exact fit triggers the
toggle to actual size. If you've scrolled even slightly, double-click resets to fit
instead. This is fine: the "already zoomed -> reset" behavior is preserved
for any manually-zoomed state.

### Other

`scale` computation moved from after the double-click block to before it (line
349), since the double-click logic now needs it. No behavior change for the
rest of the code.

### Status

Code reviewed. Behavior tested. Ready to merge.
