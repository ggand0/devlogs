# 0043 — Map art pass: river, terraced hills, vegetation (2026-07-20)

Implemented by Kimi K3 (OpenCode), directed and reviewed by the owner.

Branch `feat/map-art`. First art pass on the map since the 0005 relief
overhaul ("still not enough, but better for now"). All changes are
visual-only: no movement, morale, or combat semantics touched.

## River (`terrain.rs` + new `water.rs` + `assets/shaders/water.wgsl`)

- Meandering north-south river through the midfield, defined by three
  pure functions of z: `river_center_x` (two sines, ±165 m),
  `river_half_width` (10–18 m), `river_water_level` (0.5 + z·0.008,
  gentle fall). Same functions drive the carve, the water mesh, and the
  vegetation exclusion, so they can never disagree.
- Carve runs last in generation so the river always wins: exact bed
  profile inside the shoreline (`depth = 1.4·(1−t²)^0.75`, waist-deep
  wading at center), banks blended out to t = 2.3. Where the river
  exits through the mountainous map edge the blend cuts a gorge.
- Water: one indexed strip mesh (rows every 6 m, 9 columns spanning
  t ∈ [-1.05, 1.05] so edges tuck under the banks), custom `Material`
  with a WGSL shader. Chunky quantized vertex waves (one phase per 3 m
  cell, watertight twinkle instead of rolling swell), flat per-face
  normals from screen-space derivatives, deep/shallow banding from bed
  depth (uv.y), wobbling foam band at the shoreline, sun from a uniform
  matched to the scene light. Alpha 0.88 blend.
- Bevy 0.19 gotcha: material uniform bind group must be declared
  `@group(#{MATERIAL_BIND_GROUP})`, not a hardcoded index.

## Terraced hills (`terrain.rs`)

Heights ≥ 8 m are partially quantized into 3.5 m steps (smoothstep
ramp to 20 m, 80% pull). Flat shading turns the risers into scree
(slope > 0.75 override) → angled plateau highlands, striking around the
snowcaps. Battlefield (< 8 m) untouched.

## Ground variety (`terrain.rs`)

`band_color` now takes the triangle's world position: wheat-field
patches (fbm/70 > 0.62, gold) on low-slope grass bands + ±12% tone
wobble (fbm/45) so the open field stops reading as flat plastic.

## Vegetation (new `vegetation.rs`)

- Code-built archetypes, same cuboid/pyramid soup style as the terrain:
  pine (trunk + 3 stacked pyramids), broadleaf (trunk + 3 foliage
  cubes), bush. Per-plant yaw, scale 1.0–1.6, per-part color jitter.
- Scatter on a jittered 6.5 m grid: forest patches where fbm/100 > 0.55
  (past r > 200), bushes spill out to fbm > 0.47 (r > 150). Rejected:
  h outside 1–13 m, slope > 0.45, inside the river corridor
  (half-width·2.3 + 8 m). Pines take over above 7 m.
- Merged into one flat-shaded vertex-colored mesh per terrain chunk
  (draw calls ≈ non-empty chunks only, free frustum culling). Static;
  no instancing pipeline needed.
- ~1000 plants in 62 chunks at current tuning.

## Verification

- 200k sandbox: 76–125 fps, sim grid ~3.5–4.7 ms / step ~5 ms — same
  ballpark as pre-branch (0042). Zero render errors.
- Screenshot-compared midfield vs `main` worktree: the brown dirt
  midfield is pre-existing, not a regression.
- Armies deploy across the river and wade it (visual-only as scoped).

## Open / next

- Medieval props (watchtower, tents, fences) deferred by owner.
- River gameplay (fords, slow-down) deferred; the carve already gives
  emergent bank-slope penalties.
- Foam band + terrace step count are first-pass numbers, tune by eye.
