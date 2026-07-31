# 0002 — M2: Movement & separation (2026-07-08)

## What landed

- `src/spatial.rs` — uniform grid (cell 1.5 ≈ 2× unit radius) rebuilt every
  fixed tick via counting sort over the swarm's bounding box. Row-contiguous
  layout: a 3×3-cell disc query walks 3 contiguous `entries` slices.
- `src/movement.rs` — FixedUpdate @ 30 Hz. Double-buffered positions
  (`mem::swap`, integrate from `pos_prev` into `pos`); rendering lerps
  prev→pos with `overstep_fraction`. Integration parallelized with
  `ComputeTaskPool::scope` over 2048-unit chunks of the SoA buffers (no rayon
  needed). Per-unit: arrive-damped goal steering + separation from the grid.
- Overlap audit: full-population nearest-neighbor min/avg logged every ~2 s —
  numeric acceptance beats eyeballing screenshots.
- `FL_CAM_DIST` env var: spawn camera pre-zoomed (screenshot debugging without
  input injection — xdotool XTEST is unreliable while the desktop is in use).

## The compression saga (why naive boids stack up)

Naive `steer + separation` at 100k units converging on a goal produced
avg nearest-neighbor distance ~0.6 falling over time, min ~0.28 (cubes 0.7
wide → deep interpenetration). Three distinct causes, three fixes:

1. **Separation vectors cancel in a symmetric crowd.** An interior unit
   pushed equally from all sides feels zero net force while its goal drive
   keeps compressing the pack. Fix: accumulate *scalar* crowd density
   `Σ(1 − d/r)` (can't cancel) and fade the goal drive to **zero** between
   `CROWD_SLOW` and `CROWD_STOP`. Packed interiors genuinely stop shoving;
   rim units keep flowing. An asymptotic `1/(1+k·c)` yield was not enough —
   it must reach zero or the crowd never stops compacting.
2. **Force-level response is too slow to fix an existing overlap** at
   accel-clamped 30 Hz. Fix: positional correction (PBD-style) — pairs
   inside `HARD_RADIUS` displace apart directly (capped/tick), and the
   velocity component still driving into the correction is projected out,
   or it re-penetrates next tick.
3. **Scenario pathology**: two mirrored targets sent 50k-vs-50k head-on
   through one line → permanent crush (and my first target path moved at
   ~25 m/s vs 9 m/s unit speed — endless max-speed pursuit). Now: one shared
   lissajous target at ~unit speed per the spec.

Result: avg NN stable at **0.74–0.78** over 80 s (cube width 0.62 → visible
gaps), min ~0.43 (rare transient close pairs in shear zones, invisible at
scale). No compaction drift.

## Perf & numbers

- Sim tick: grid rebuild ~1.5 ms + step ~4 ms at 30 Hz (12-core Ryzen 3900X,
  opt-dev profile) → ~5.5 ms every other frame.
- Frame: **222 fps / 4.9 ms** when the compositor doesn't throttle; unit draw
  still ~1 ms GPU.
- `opt-dev` cargo profile added (release minus LTO, 16 codegen units) — dev
  profile's opt-level=1 sim was noticeably slower.

## Emergent behavior worth keeping

Teams don't interpenetrate when converging: they stratify into side-by-side
lobes / a two-tone comet chasing the target, with lane formation when streams
cross. Pure separation + yield, no team-awareness in the code.

## Look

Cubes slimmed to 0.62×0.9×0.62 — soldier proportions, and at-equilibrium
spacing (~0.78) reads as a tight formation with daylight between units
instead of a merged mass.
