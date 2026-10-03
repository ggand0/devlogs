# Big Grassland implementation and first playtest
Written by GPT-6 Astra.

2026-10-03. Implementation at `809ef2a` on `feat/big-grassland`, based on `main` `25ec337`. This entry records the code details for Claude's review and Gota's feedback after playing the first version. It supplements [0179](0179-big-grassland-first-playable-field.md), which records the initial implementation and validation. The graphics working tree is clean; this branch has not been pushed or merged by Astra.

## Playtest result and scope

Gota tried Big Grassland and reported that it looks good, with no performance issue so far in the default army layout. His observation is that the soldiers spread thinly across the larger field and most render at L2 or L3. He will have Claude check the implementation. This is feedback from the default layout, not a measured conclusion about dense melee, a close camera, or larger armies.

The first playable stage is complete. It adds the larger field, initial landforms, the existing pasture surface and camera support. Peripheral planting, refinement of the landforms, the proposed deployment layout and controlled terrain/shadow comparisons remain separate work. No further implementation was requested or performed in this documentation turn.

The approved design is [note 030](../work/notes/030-big-grassland-proposal-2026-10-03.md); the first implementation handoff is [note 031](../work/notes/031-big-grassland-first-field-2026-10-03.md). The original brief is [handoff 086](../work/handoffs/086-big-map-for-astra-2026-10-03.md).

## Map selection and dimensions

`src/terrain.rs` adds `MapKind::BigGrassland`, the menu label `Big Grassland`, and the launch value `FL_MAP=big_grassland`. `MapKind::ALL` has five entries, with the new field immediately after Grassland. The menu already builds its map buttons from that array, so no game-state or menu code needed editing. Grassland remains the default.

Saved setups serialize the new variant as `map: BigGrassland`. The existing setup precedence is preserved: a loaded setup, including the built-in Grassland demo, selects its own map before `FL_MAP` is considered.

| Property | Grassland, Classic, River, Sandbox | Big Grassland |
|---|---:|---:|
| Playable width by depth | 1024 × 768 m | 2048 × 2048 m |
| Bounds on x | -512 to +512 m | -1024 to +1024 m |
| Bounds on z | -384 to +384 m | -1024 to +1024 m |
| Height cell size | 2 m | 2 m |
| Cells per chunk axis | 32 | 32 |
| Chunk grid | 16 × 12 | 32 × 32 |
| Height vertices | 513 × 385 | 1025 × 1025 |
| Chunk count | 192 | 1024 |
| Terrain triangles | 393,216 | 2,097,152 |

`MapKind::chunk_counts()` supplies the chunk dimensions. Its `grid_verts()` derives the height-grid dimensions, and `half_extents()` exposes the map's half width and depth. The former global `CHUNKS_X`, `CHUNKS_Z`, `VERTS_X`, `VERTS_Z` and `HALF_EXTENTS` are removed from production code. Fixed dimensions remain only in the old-size test fixtures.

Each `Terrain` stores `verts_x` and `verts_z` beside its height and blocked arrays. Its public `grid_verts()` returns the actual dimensions; its chunk counts derive from them. Height indexing remains row-major, `z * verts_x + x`. Bounds, bilinear sampling, blocked-mask lookups, crater edits and dirty-chunk indexing all use the instance's dimensions. The four old maps retain their height formulas, origin calculations, sampling arithmetic and blocking/wading rules.

## Terrain meshes and map switching

The mesh builders still use 2 m cells and the existing alternating triangle diagonals. Heightfield normals sample neighbouring vertices across chunk boundaries. Crater invalidation still includes the neighbouring vertices needed to rebuild those normals. Coverage-image dimensions, material coverage bounds and original-height lookups also use the active terrain stride.

Map switching needed more than changing the height vector. The old implementation reused a fixed set of chunk entities; that would leave most of the larger field undrawn, or leave extra ground when switching back. `rebuild_map()` now builds the new terrain, replaces the original-height snapshot used for crater soil, creates the appropriate material, despawns the old chunk entities and clears the retained mesh handles. `spawn_chunk_meshes()` creates the full new set of meshes, entities, materials and bounding boxes. Startup uses that same helper. Dirty-chunk rebuilding remains the path for subsequent crater edits.

Classic keeps its flat-shaded band material. The grassland maps use the existing authored pasture material, and River retains its existing procedural coverage. The texture handles kept warm across map switches are unchanged. There are no new texture files, shader changes, render distance levels or coarser simulation cells in this commit.

## Landforms and ground appearance

`big_grassland_height()` is separate from the old Grassland height function. Its central surface is:

```text
5 + 1.5 cos(x / 340) cos(z / 420)
  + 0.7 sin(x / 210) cos(z / 300)
```

Coordinates and heights are in metres. The cosine terms along z give the two deployment sides the same central relief. Outer landforms use smoothstep transitions beginning beyond x or z = ±850 m, multiplied by broad exponential falloffs. The western hill is centred near (-950, -350), with a 23 m contribution; the eastern shoulder is near (+970, +200), with a 14 m contribution. The north and south edges have smaller 6 m and 4 m contributions. These are height-function contributions, not measurements of the final summit above every surrounding point.

Across the open 1700 × 1700 m square, the test samples every 2 m. Heights range from 3.247 to 6.824 m, a 3.577 m spread. The largest sampled grade is approximately 0.00755, or 0.755 percent, below the proposed 3 percent limit. The open area has no blocked vertices.

The existing `grassland_layout_color.ktx2` is mapped over the larger square. Its broad colour pattern therefore covers more ground; the shader's fine grass sampling remains in world metres. This stage does not contain newly authored ground artwork. `src/vegetation.rs` has an empty planting branch for Big Grassland, so it has no trees or shrubs yet. The planting on the four existing maps is untouched. Water is disabled through the existing non-river behavior; no water code was edited.

## Camera, picking and selection rings

`src/camera.rs` gives Big Grassland a 2800 m maximum camera distance and a 6000 m far plane. The proposed 2200 m distance was insufficient to fit the full square vertically at the current field of view. Other maps retain their 900 m zoom limit and default 1000 m far plane. When terrain changes, camera distance and target distance are clamped to that map's limit and the perspective far plane is updated. The existing scripted camera sweep uses the active map's maximum distance.

Camera focus remains clamped to the terrain bounds and follows its height. The camera eye can still sit beyond the playable edge, and the finite terrain edge is visible. Distant landscape and camera-eye containment are not implemented.

`Terrain::raycast()` retains 1.5 m march steps and ten bisection iterations. Big Grassland permits 4000 steps, approximately 6 km of reach; old maps retain 1500 steps, approximately 2.25 km. This supports ground picking from the larger overview without coarsening the hit calculation.

The narrow shared-file change in `src/selection_rings.rs` makes `update_ring_terrain()` read `terrain.grid_verts()` instead of the removed global function. The shader already receives origin, cell size and vertex dimensions as uniforms, so it needed no edit. The existing delayed upload behavior is preserved: a terrain change marks the ring heightfield stale, and the next active selection refreshes the heights and dimensions together.

In `src/battle_setup.rs`, the three `HALF_EXTENTS` references become `MapKind::Grassland.half_extents()`. They belong to the fixed Grassland demo and its test, so this preserves that demo's placement. Normal deployment already asks the active Terrain for its bounds.

## Validation already completed

The implementation passed an offline opt-dev build, strict clippy with `--all-targets -- -D warnings`, and all 35 tests. Six added tests cover map dimensions and corner indexing, the open-ground slope and overview picking, far-corner crater updates and mesh seams, repeated small/large map switching with crater rebuilding, camera range/bounds after switching, and selection-ring refresh after switching while nothing is selected. Existing saved-setup and Grassland demo tests also pass.

The deterministic comparison used `work/scripts/hashrun.sh` and `hashcmp.sh`, with `FL_TEST_DIR=1`, on each original map before and after the change. All 13 comparable fingerprints through tick 780 match on each map: 52 of 52 total. This establishes equivalence for those scenarios and ticks; it is not a dense-battle performance test. Logs and hashes are under `tmp/runs/scripts/big-grassland-{before,after}-<map>.*`. The untouched baseline binary and terrain source are under `work/baselines/big-grassland-2026-10-03/`.

Runtime checks included 200,000 standing soldiers with `FL_ARMY_GAP=500`, and a saved Big Grassland setup placing two 1000-man units at (800,-750) and (-800,750), beyond the old map bounds. The latter correctly selected Big Grassland even with `FL_MAP=grassland`. No shader errors or panics appeared in the capture logs. The user's existing saved setup was not edited.

Review captures are under `tmp/shots/big-grassland-v1/`: `overview_7s.png`, `armies-200k_8s.png`, `west-hill_7s.png`, `saved-setup_7s.png` and `zoom-and-craters.mp4`. The recording contains continuous zoom/orbit and repeated crater edits. A time-ordered frame sequence was inspected for missing chunks and seams; no full-rate flicker assessment or controlled terrain/shadow timing comparison was claimed.

## What remains for Claude to check

Normal spawning still fills the map width and uses the global default army gap of 60 m. The planned two 1400 × 600 m deployment areas, 500 m default separation and 150 m clear flanks are not encoded in the spawn/deployment logic. `FL_ARMY_GAP=500` was an explicit option for the capture, not a new default. Gota's playtest observation about thinly spread armies is consistent with this unchanged packing rule.

The spatial-grid rebuild and density field still have their existing area-dependent costs. No sim source, unit renderer, unit shader, soldier scale, formation spacing or level-of-detail threshold changed. Claude's review can compare this diff against `25ec337` and then check the spawn/deployment integration, dense formations and close-camera cases before drawing performance conclusions. The 200k benchmark remains intact; 300k to 500k benchmarks, grid work and fast-forward remain the separate work described in handoff 086.

For the next graphics stage, add peripheral planting and review the edge landforms. Measure terrain and shadow costs at matched views before choosing any mesh-detail reduction. Gota's initial playtest gives no reason by itself to change soldier detail thresholds or enlarge the simulation cells.
