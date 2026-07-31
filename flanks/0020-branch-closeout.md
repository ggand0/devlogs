# 0020 — m2tw branch closeout: restart key, SIMD negative result (2026-07-10)

Closing `feat/m2tw` for PR review (`ee73326`, refactor commit after).

## Crumbs

- **R restarts the battle**: fresh armies, cleared kills/fled/selection/
  outcome — play-test iteration without relaunching. Terrain untouched
  (craters are disabled anyway).
- **F stance toggle deleted** along with the `Stance` enum: it was a dead
  control (nothing consumed the value since M5). Formation shapes return
  as real code in the rigid-formations milestone.

## PR6 (SIMD scan kernel): built, measured, REVERTED

The plan's stretch item — split the grid's cell-ordered units into
x/z/meta lane arrays and process the fused neighbor scan (separation +
targeting) 8 candidates at a time with `wide::f32x8`.

Three iterations, all measured at 200k on the 3900X:

| Variant | battle step | idle step |
|---|---|---|
| scalar baseline (shipped) | 6.3–8.0 ms | ~2.5 ms |
| SIMD, masked partial blocks | 6.0–6.8 | 4.2–4.7 |
| SIMD, full blocks + scalar tails + hoisted splat consts | 6.4–8.8 | 3.4–5.6 |

Why it loses:
1. **Lane occupancy**: a 3×3-cell query yields three row-runs of ~3–15
   units. Sparse crowds → almost everything is a masked block or a
   scalar tail; dense combat → barely 1–2 full blocks per row.
2. **Cache lines**: splitting one 16-byte `SortedUnit` stream into three
   arrays turns 1 cache-line touch per short run into 3. The short-run
   majority pays this on every query — that's the idle regression.
3. **Amdahl**: the scan is only ~half of `step`; terrain sampling, goal
   steering, the swing state machine, and integration are scalar and
   untouched.

Reverted to the scalar scan over AoS `SortedUnit`s (bit-identical sim
code to before; comments document the result). `wide` dependency removed.
**Do not retry** without (a) an AoSoA block-of-8 layout so scalar AND
vector paths share cache behavior, and (b) vectorizing the rest of the
integrate body — and only when scale actually demands it.

## Refactor sweep before the PR

- `util::env_or<T>()` replaces six hand-rolled env-parse blocks
  (FL_UNITS / FL_REG_SIZE / FL_HEAVY_FRAC / FL_COMBAT_SCALE / FL_CAM_*).
- Stale `#[allow(dead_code)]` dropped from `UnitTypeParams` (every field
  is read now).
- Clippy: zero warnings across the branch.
