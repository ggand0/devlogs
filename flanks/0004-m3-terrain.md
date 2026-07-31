# 0004 — M3: Deformable terrain (2026-07-08)

## What landed (`src/terrain.rs`)

- **Heightfield**: one 513×385 vertex grid (2 m cells → 1024×768 m world),
  rendered as 16×12 chunk entities of 32×32 cells each. Generation: 4-octave
  value-noise FBM (hash-based, no deps), amplitude ~11 m, center flattened so
  the battlefield is walkable and hills frame the outskirts; +1.4 m bias puts
  the flat center in the grass color band instead of dirt.
- **Look**: flat-shaded triangle soup (per-face normals, no index buffer),
  one hard height-banded color per triangle (crater dirt → dirt → grass →
  highland → rock), steep faces override to scree gray, quad split diagonal
  alternates per cell. StandardMaterial with vertex colors, roughness 1.
- **Units on terrain**: bilinear `height_at` snaps Y in the sim integrate
  loop; `slope_at` (finite-difference gradient) applies speed penalty
  `1/(1+3·slope²)`. Craters visibly slow/pool units — nice emergent hazard.
  Position clamped to the field. Camera focus follows terrain height.
- **Craters**: heightfield subtraction, smooth bowl `(1−(d/r)²)²` plus a
  raised rim ring (12% of depth out to 1.35r) for readability. Affected
  chunks marked dirty; `remesh_dirty` rebuilds them the same frame and
  updates their `Aabb` (bevy does NOT recompute Aabb when a mesh asset is
  replaced — stale bounds would break culling).
- **Tools**: `X` carves under the cursor (ray-march + bisection against the
  heightfield). `FL_TEST_CRATERS=1` auto-carves near the center every 2 s —
  perf/visual verification without input injection.

## Acceptance

- Craters appear instantly (same-frame remesh; a crater touches ≤4 chunks,
  each ~6k verts — sub-ms rebuild).
- FPS holds: 80–107 fps with 20 craters carved over 40 s, sim tick now
  grid ~1.7 ms + step ~6 ms (slope sampling added ~2 ms; still fine at 30 Hz).
- Armies flowing over hills and dipping into crater bowls at 100k units.

## Gotchas for later

- `Mesh::compute_aabb` moved to the `MeshAabb` trait (bevy_camera) in 0.19.
- Slope penalty makes craters sticky (units slow inside). When combat's
  crater-on-death lands (M6), watch for units pooling in shell holes —
  might be a feature (cover), might need tuning.
