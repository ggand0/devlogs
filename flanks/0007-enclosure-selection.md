# 0007 — Enclosure (lasso-loop) selection (2026-07-08)

## Owner request

"When I enclose the line in a circle, select the entire thing inside — like
War of Dots selection."

## Implementation (`src/orders.rs`)

- **Closed-stroke detection**: a stroke counts as a loop when its endpoints
  are within `clamp(15% of path length, 10..40 m)` of each other — forgiving
  enough that a sloppy circle closes, tight enough that a straight line
  never does.
- **Polygon test**: standard even-odd crossing test against the stroke
  (closing edge implied). Selection is the **union** of inside-the-loop and
  within-14 m-of-the-stroke, so open strokes behave exactly as before and a
  loop also grabs the units under the pen line.
- **Feedback**: while dragging, the implied closing edge renders as a green
  gizmo segment as soon as the stroke would close — you can see the loop
  "lock in" before releasing.
- Perf: bbox early-out then ~n_edges ops per candidate unit, one-shot on
  release; negligible.

## Verification

`FL_TEST_ORDERS=1` now drives selection through a synthetic 28-point circular
stroke (the real polygon path, not a shortcut): selected 5540 units =
3245 strictly inside the r=45 circle (brute-force cross-check logged) plus
~2300 in the 14 m stroke band. Union semantics confirmed.
