# 031 — PR #22 and #23 review (YelovSK)

Both PRs are from YelovSK (Patrik Hampel), a first-time contributor. Self-described
as having zero Rust experience, using AI for syntax. Both touch only `src/pane.rs`.

## PR #23 — Use smooth scroll and align wheel zoom input (+6 / -7)

### The bug

Plain scroll zoom (mouse_wheel_zoom enabled, no Ctrl) feels choppy compared to
Ctrl+scroll zoom. Both go through `zoom_image()` but read different inputs:

- **Ctrl+scroll**: egui consumes the scroll events and produces `zoom_delta()`,
  which internally reads `smooth_scroll_delta` — egui applies exponential
  smoothing over multiple frames before returning the value.
- **Plain scroll**: our code reads `raw_scroll_delta.y` — discrete wheel ticks
  with no smoothing, so each notch is a single jump.

The smoothness difference has nothing to do with Ctrl itself. It's just that
`zoom_delta()` happens to use the smoothed input, while our plain scroll path
was reading the raw input.

### What the PR does

1. Switches `raw_scroll_delta` to `smooth_scroll_delta` — plain scroll now gets
   the same frame-by-frame interpolation that Ctrl+scroll already had.
2. Removes the `if scroll != 0.0` guard — unnecessary because `(0.0 * 0.003).exp()`
   equals 1.0, which is the identity for multiplication.
3. Rescales `zoom_delta()` (pinch) from egui's default speed (0.005) down to 0.003
   so both zoom paths use the same speed factor.

### Review concern: speed choice

The contributor describes 0.003 as the "rough" zoom speed and 0.005 as the
"smooth" one, but then unifies everything at 0.003. This means Ctrl+scroll
zoom gets slower than before to match the plain scroll speed, rather than
bringing plain scroll up to the Ctrl+scroll speed.

Worth asking: why not unify at 0.005, or make it configurable?

### egui internals (for reference)

`smooth_scroll_delta` is documented at `InputState` in egui:
https://docs.rs/egui/latest/egui/struct.InputState.html

The smoothing logic lives in `egui/src/input_state/mod.rs` (lines 511-541 in
egui 0.33.3). It uses `exponential_smooth_factor(0.90, 0.1, dt)` to spread
large discrete scroll ticks across multiple frames. Small deltas (< 1.0) pass
through unsmoothed.

When a zoom modifier (Ctrl/Cmd) is held, egui routes scroll events into a
separate `smooth_scroll_delta_for_zoom` accumulator and bakes it into the
`zoom_factor_delta` returned by `zoom_delta()`. The scroll_zoom_speed option
(default 1/200 = 0.005) controls the scaling. Our code never sets this, so it
uses the default.

Related egui issues:
- https://github.com/emilk/egui/issues/4777 (Ctrl+scroll zoom regression)
- https://github.com/emilk/egui/issues/7650 (scroll to zoom without Ctrl in Scene)

### Status

Needs testing. Checkout with `gh pr checkout 23`.

---

## PR #22 — Toggle actual size on double-click (+18 / -4)

### What it does

Double-click toggles between fit-to-screen and actual size (1:1 pixel zoom).
Standard image viewer behavior. When zooming to actual size, it zooms toward
the cursor position. When zooming back to fit, it resets pan.

Currently double-click resets zoom and pan unconditionally (lines 355-358 in
`pane.rs`). The PR replaces that with a toggle.

### Status

Not reviewed yet. Larger change, more logic to verify (zoom-toward-cursor math,
interaction with existing zoom/pan state). Review after PR #23.
