# 0008 — M5: Frontline solver (2026-07-08)

## Architecture (`src/frontline.rs`)

All 2D, all coarse, once per fixed tick:

1. **Influence fields**: per-team density splatted into an 8 m grid
   (128×96), 2 passes of separable radius-1 box blur. Bilinear `sample` +
   central-difference `grad`.
2. **Per group**: centroid refresh; facing = order direction, else enemy
   density gradient; engaged = enemy influence above threshold at centroid
   or 30 m ahead. Width/depth from stance: Hold = 10 rows (wide), Column =
   50 rows (narrow); `half_width = count/rows · spacing / 2`.
3. **Front curve**: 48 lateral samples; each marches −24..+72 m along
   facing on the coarse field to the first point where enemy influence
   exceeds ours (balance point); no contact → leading-edge default (+14 m).
   Two lateral smoothing passes.
4. **Neighbor linking**: per team, engaged groups sorted along the team
   lateral axis; adjacent curves' closest endpoint pairs (within 120 m)
   blended toward their midpoint over the outer 6 samples. Links recomputed
   every tick, so cut/destroyed groups re-link or leave a gap automatically.

## Unit steering (movement.rs, engaged branch)

No explicit slot list. Each engaged unit projects onto the group's lateral
axis, lerps the front curve at that u, and steers to
`curve(u) − facing · clamp(depth·0.85, 0.7, max_depth)`. The continuous
forward pull + crowd-density yield produce ranks at the front and a reserve
block behind (deep units cap at max_depth); gaps refill because the pull
never stops. Overextended units (ahead of the curve) get pulled back.

Group data is flattened into a snapshot (curves in one buffer + per-group
ranges) for the parallel integrate loop.

## Emergent behavior

- Two ordered armies collide → sharp stable two-color interface, front
  curves tracing the whole contact band; the standoff gap comes from
  cross-team separation + crowd yield (combat in M6 will consume it).
- Balance-point extraction means local numerical superiority moves the
  curve — pressure and pushing are emergent, nobody codes "advance".
- Engaged armies at full contact: step ~4.4 ms (the feared M4
  interpenetration cost never happens — fronts prevent deep overlap).

## Acceptance

- Stable visible battle line between two armies: ✓ (screenshots, 139–280
  fps, line held for minutes).
- Cut group protrudes as salient on order: ✓ (scripted 5.3k-unit cut pushed
  through; owner also live-played during the test run — lassos, orders and
  extra cuts all behaved).
- Stances: F toggles Hold(10 rows)/Column(50 rows) on selected groups —
  implemented; visual A/B still worth a dedicated check later.

## v2 rewrite: ONE line (owner-reported, second attempt)

Owner: per-group curves still produced multiple lines after cuts — "you
only need a single line". Correct: the per-group-curve model (each group
extracts its own front, endpoints stitched) was structurally wrong for
overlapping groups. Replaced with a shared-contour model:

- **THE front = the phi = 0 contour** of `phi = blue_density − orange_density`,
  extracted by marching squares on the 8 m grid, masked to cells where both
  teams are present. One battle → one line; disjoint battles → one line each.
  Both teams share it by definition.
- **Nearest-front lookup**: multi-source BFS from the contour segments gives
  every cell within ~60 m its nearest front point + front normal. Units near
  the front hold `clamp(depth·0.85, 0.7, stance_depth)` behind their nearest
  point on their own side (normal sign flips per team); everyone else follows
  group orders. Per-UNIT engagement — no group curve state at all.
- Groups slimmed to: order, stance (→ depth cap), centroid, engaged flag.
  Deleted: per-group curves, widths, facing, axis, endpoint stitching.

Marching squares + BFS on 129×97 cells is sub-ms. Verified: single
continuous line across the whole battle; salient press dents the line
locally (5k extra units vs a 50k line = small bulge — believable physics).

## Bugfix v1 (superseded by the v2 rewrite): phantom front lines

Sending a selection created "multiple lines": group front width came from
`count/rows`, so big groups claimed ~800 m fronts — flank rays over empty
ground emitted default-advance points → long phantom curve segments, and
every cut added another overwide, jittering curve (raw gradient facing).
Fixed: width = `min(stance width, 2.2·σ of lateral unit spread + 20 m)` so
curves hug the actual mass (the +20 m margin still lets Hold stance widen
the formation over time), and facing is temporally smoothed (lerp 0.2/tick).

Known remaining artifact: a group whose units sit in two far-apart clusters
(scattered lasso) still draws one curve spanning the gap — "one group, one
front" by design. Fix if it matters: split disconnected clusters.

## Notes

- Engaged groups never "arrive" (blocked), so orders persist as pressure —
  clear-on-arrival now skips engaged groups.
- NN min ~0.33 under press (transient tight pairs at the line); fine
  pre-combat, revisit with M6 kills.
