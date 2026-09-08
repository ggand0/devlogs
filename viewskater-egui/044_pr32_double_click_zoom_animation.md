# PR #32: Animate zoom/pan on double-click

PR: https://github.com/ggand0/viewskater-egui/pull/32
Author: YelovSK (Patrik Hampel)
Merged: 2026-09-08 (main e7763db). Branch: two contributor commits, a merge of main
(0806e25), and one comment-only commit from our side (eca7cf2).

## Summary

Two independent changes, both in `pane.rs`, plus a new 79-line `src/view_animation.rs`.

1. Custom double-click detection replaces `response.double_clicked()`. Two primary
   clicks within 0.30 s and 8 px on the same image count as a double-click. YelovSK's
   reason: egui's detector treats a quick third and fourth click as triple-clicks, so a
   second double-click is swallowed until about 0.6 s has passed.
2. The fit-to-screen / 1:1 toggle on double-click is animated over 0.12 s with an
   ease-out cubic instead of jumping. Target computation is unchanged from main: from
   fit, go to 1:1 anchored at the click when the image is larger than the pane, 1:1
   centered otherwise; from anything else go back to fit.

`view_animation.rs` holds `ViewTransform` (zoom + pan), `ViewAnimation` (from, to,
start time, duration, easing) and `sample(now)` which returns the interpolated
transform and a done flag. No threads, no shared state. Panes stay in sync because
each reads the same frame clock. egui 0.31 has no public API for a tweened transition
between two coupled values; `Context::animate_value_with_time` is linear, single
value, fixed duration. `emath::easing::cubic_out` does exist in 0.31 and is identical
to the `Easing::EaseOutCubic` he wrote, so the enum could be dropped later.

## How it sits in show_image

The animation touches `show_image` in four places, which is what made it look tangled
at first read:

- top of the function: `advance_view_animation(now)` applies a running animation to
  zoom/pan every frame and requests a repaint until done
- drag and scroll zoom: cancel a running animation (one assignment each); drag also
  clears the pending first click
- double-click block: computes the target and stores a `ViewAnimation` instead of
  assigning zoom/pan; the image draws at the old transform on that frame and starts
  moving the next frame
- draw: unchanged, reads the final zoom/pan

The state is one `Option<ViewAnimation>` field on `Pane`; nothing else reads it.

Our comment commit numbers these as steps 0 to 4 in the function body and restores a
doc comment stating the order and the rule that steps 1 to 3 write zoom/pan and step 4
reads them. No code changes; verified with `git diff -w` that every changed line is a
comment. It also rewords a pre-existing clip comment.

`zoom_target` (anchored zoom maths, pure geometry on zoom/pan/anchor/rect) was moved
from inline scroll-zoom code into a `Pane` method and reused for the double-click
target. It could live on `ViewTransform` instead, making the module a coherent "view
transform and its transitions" unit. Noted for a future refactoring round, not asked
for in the PR.

## Merge with main

The branch forked at b7d1154 (June) and was 74 commits behind. Four merged PRs had
touched `pane.rs` since: #25 reset zoom/pan on navigation, #18 slider preview, #31
discovery options, #28 animation player. Five conflict hunks, four of them both-sides-
additive (imports, struct fields, constructor, helper methods). The fifth was the
double-click else branch where main calls `reset_view()` and the branch uses the
animation target (1.0, ZERO); these are the same thing, kept the branch. Main's diff to
`pane.rs` since the fork is +103/-28; the merge commit's diff against the branch tip is
+102/-26, the two-line difference being that else branch. Build, 6 tests, clippy clean.

## Owner testing (Ubuntu)

- Compared against main: the animated toggle feels smoother and better than the jump.
- Repeated double-clicks land on the same zoom position.
- Quick successive double-clicks register; on main the second one is swallowed.
- Cancel-by-drag mid-animation is hard to trigger by hand at 120 ms; assumed fine
  from the code.
- Dual pane animates in sync.
- Navigating mid-animation: nav takes over and resets zoom. In the code there is a
  theoretical window where a key press inside the 120 ms would let the animation
  finish on the new image, because `reset_view` does not clear `view_animation`. Not
  reproducible by hand; not fixed.

## Behaviour differences vs main worth knowing

- 1:1 zoom is no longer clamped to MIN_ZOOM/MAX_ZOOM on the centered path (image
  smaller than the pane). Only matters for an image more than 20x smaller than the
  pane, where `1/scale` drops below MIN_ZOOM 0.05. Main clamped it. Not fixed.
- Owner noticed zoom does not persist while holding A/D. That is main behaviour from
  #25: "Reset Zoom/Pan on Navigation" defaults to true and calls `reset_view` on every
  navigation step. Unrelated to this PR; owner may adjust in a separate PR.

## Naming

`ViewAnimation` animates a `ViewTransform` (the viewport zoom + pan), consistent
within the module. `ViewTransition` or `ZoomPanAnimation` would be alternatives; no
rename requested.
