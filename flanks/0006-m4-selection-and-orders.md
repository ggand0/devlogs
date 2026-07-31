# 0006 — M4: Groups, selection & orders (2026-07-08)

## What landed (`src/orders.rs`)

- **Groups**: `Groups` resource — `{team, order: Option<Vec2>, count}` per
  group; units carry a group index (`Units::group`). Two initial army groups.
  Movement reads per-group orders: `Some(target)` = attack-move with arrive
  slowdown + slope penalty; `None` = hold (separation only). Groups auto-clear
  their order when the group centroid gets within 18 m of the target.
  (Kept as a resource, not per-group entities — M5 will decide if front-curve
  state wants entities; no speculative abstraction now.)
- **Selection**: hold LMB and draw a line — cursor is raycast onto terrain
  each frame into a ground polyline (gizmo-visualized while dragging);
  on release, friendly units within 14 m of the polyline are selected
  (bbox early-out + point-segment distance; brute force over 100k is fine as
  a one-shot). Selected units get a yellow-white tint via per-instance color.
- **Orders**: right-click = attack-move the selection. If the selection is a
  strict subset of a group it is split into a new group first (M5 replaces
  this with attached-salient behavior per the spec; the split keeps orders
  group-level meanwhile). `C` = explicit cut, no order.
- **Verification**: `FL_TEST_ORDERS=1` scripted run — selects a 45 m disc of
  the blue army (3245 units), cuts, sends it across the map; later orders the
  remaining 47k west. Screenshots show the highlighted lens detaching and
  leaving a bite-shaped gap, then the armies traveling as independent bands.
- Overlay: group count + selected count.

## Acceptance

Carve a sub-blob out of a swarm and send it elsewhere: ✓ (scripted run,
logs + screenshots; interactive path shares the same code minus the input).

## Perf note for M5/M6

Deliberately marching 47k blue THROUGH 50k orange (bad first test script)
doubled local density and pushed the sim step to ~15.5 ms (27 fps) — deep
crowd interpenetration is the engine's worst case. M5 fronts exist precisely
to prevent this state (contact bands instead of full overlap), and M6 kills
units in contact. Keep an eye on it; if needed, the separation loop has
headroom (SIMD, cheaper kernel, 2-tick neighbor caching).

Idle armies are nearly free (step ~1.8 ms): slope sampling now only runs for
units that have an order, and holding units have zero goal drive.
