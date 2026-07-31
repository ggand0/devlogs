# 0010 — Prototype complete: architecture capstone (2026-07-08)

All six milestones done in one day. 2,499 lines of Rust + WGSL, 10 commits,
Bevy 0.19. This entry is the return-after-a-month orientation doc; details
live in entries 0001–0009.

## The one-paragraph pitch that survived contact with implementation

Two 50k armies on deformable terrain. Units live in SoA buffers, never as
entities. The player draws lassos, cuts groups, orders attack-moves; armies
form a single emergent battle line where their influence balances, grind
each other down, and the accumulated death craters dig a canyon along the
front. War of Dots / Minecraft look: instanced cubes, flat shading, hard
height-banded colors.

## File map (src/, ~250 lines each except orders/terrain)

| File | Owns |
|---|---|
| `units.rs` | SoA store: pos/pos_prev/vel/speed/team/group/hp/color; army spawn; `hash01` |
| `render_units.rs` | Custom instanced pipeline (1 draw call), per-frame instance upload, tick interpolation, selection tint |
| `shaders/unit_instancing.wgsl` | Embedded shader: instance pos+scale+color, per-face lambert |
| `spatial.rs` | Counting-sort uniform grid (1.5 m cells) over swarm bbox, row-contiguous 3×3 queries |
| `movement.rs` | 30 Hz FixedUpdate step: goal/slot steering, separation + crowd yield + PBD correction, melee damage gather, parallel over SoA chunks |
| `terrain.rs` | 513×385 heightfield, 16×12 flat-shaded chunk meshes, crater carve + same-frame remesh, ray-march picking |
| `frontline.rs` | Per-team density fields (8 m grid), phi=0 marching-squares contour (THE line), BFS nearest-front lookup, test scripts |
| `orders.rs` | Groups (order/stance/centroid), lasso + enclosure selection, attack-move/cut/stance keys |
| `combat.rs` | Death sweep: swap-remove all columns, selection sync, kill stats, death craters |
| `overlay.rs` | FPS/sim/GPU-timing overlay + periodic log lines (the verification backbone) |
| `camera.rs` | RTS camera, terrain-following focus, FL_CAM_DIST |

## Data-flow per fixed tick (30 Hz)

density splat+blur → phi contour (marching squares) → BFS front lookup →
group centroids/engagement → swap(pos,pos_prev) → grid rebuild (counting
sort) → parallel integrate (steer + separate + fight) → death sweep
(swap-remove) → order arrival clear. Rendering interpolates pos_prev→pos
by overstep fraction every frame.

## The five hard-won lessons

1. **Trust GPU pass timings, not FPS** — this desktop's compositor clamps
   presents to ~60 Hz seconds after launch (0001).
2. **Separation vectors cancel in symmetric crowds**; scalar density that
   fades goal-drive to zero is what stops self-compression, plus PBD
   positional correction + approach-velocity kill for existing overlaps (0002).
3. **Bevy doesn't recompute Aabbs on mesh-asset replacement** — refresh
   manually on remesh or culling breaks (0004).
4. **One battle = one line**: per-group front curves are structurally wrong
   (overlapping groups → multiple lines). The shared phi=0 contour gives a
   single line by construction and deleted more code than it added (0008).
5. **Fronts are also the perf fix**: they prevent crowd interpenetration,
   which was the engine's worst case (~15 ms step); fully engaged 100k
   ticks at ~4.4 ms (0006, 0008).

## Final numbers (RTX 3090 + Ryzen 3900X, opt-dev, desktop contended)

- Unit draw: ~1 ms GPU at 100k (one draw call). Total GPU frame ~1.6 ms.
- Sim tick at full engagement: ~4.4 ms step + ~2 ms grid + <1 ms fields.
- Peak combat churn ~1,400 kills/s; battle 100k → 3.6k in ~5 min (FL_DPS=120).
- 405 fps / 2.5 ms when the compositor doesn't interfere.

## Controls & debug

WASD pan, wheel zoom, middle-drag rotate. LMB-drag lasso (close the loop to
select inside), RMB attack-move, C cut, F stance, X crater, G gizmos.
Env: FL_CAM_DIST, FL_DPS, FL_TEST_CRATERS, FL_TEST_ORDERS, FL_TEST_FRONT.

## Backlog (owner-flagged or deferred)

- Terrain drama pass #2 ("still not enough") — cliffs/plateaus/canyons?
- Endgame: rout/morale so battles truly finish; remnant group merging.
- Split disconnected clusters into separate front behavior (minor).
- Total War pivot remains open: per-unit facing + rigid slot grids swap in
  behind the group abstraction (0003).
