# 0160: tree shadows, the cascade range, and a perf plan for static shadows

Written by Claude Opus 5.5.

2026-09-29. Follows 0159 (main merged into feat/vegetation, sun shadows on the oaks). Diagram: work/notes/vis/024-shadow-casters-per-category.png (script work/scripts/viz/castersviz.py).

## What the screenshots showed

Gota's captures in the Sandbox scene, all in resources/vegetation/:

- debug13.png: close view, shadows on. Ground shadows under every oak and self-shaded canopies. The trees read far better than without shadows.
- debug14.png: mid distance with the old default, 2 cascades to 110 m. Trees inside 110 m keep their ground shadow and dark canopy; the ones past it have neither and read pale. The third tree from the left is dark in debug13 and pale here.
- debug15.png: `FL_SHADOW_CASCADES=3 FL_SHADOW_DIST=280`, trees farther out than in debug14, all shaded.
- debug16.png: same settings, farther still: past 280 m the trees go pale again. The pop moves out, it does not go away.

debug15 against debug14 settles the cause of the paleness: the missing canopy self-shadow, not the detail level. The oak switches L0 to L1 below 108 px on screen and to the card below 16 px (`select_tree_level`). For a full-size oak (about 13.5 m) in Gota's window at Bevy's default 45 degree field of view that is about 140 m and 950 m; planted oaks are scaled, so each tree differs. The card never falls inside the shadow range.

## How the cascades and the categories relate

A cascade is one shadow map belonging to the sun, covering a band of view depth. Everything that casts in that band is drawn into it, and every pixel in the band reads it. There is no cascade count per object category. What can differ per category is which maps a category is drawn into, which comes down to how far it casts:

| Category | Casts into | Notes |
|---|---|---|
| Soldiers | cascades 0 and 1 | Only soldiers the camera draws at L0 or L1 cast (`CAST_LODS`), which ends before cascade 2 starts at 85 m. 0145 counted zero soldiers in cascade 2. Chosen per cascade on the GPU. |
| Trees | every cascade | Bevy `StandardMaterial` meshes, foliage `AlphaMode::Mask(0.5)`. |
| Terrain | every cascade | At the current sun (yaw 0.7, pitch -0.75) the hills shade nothing (0145). Drawn anyway. |
| Banners, rings, bars | none | `NotShadowCaster` since PR #12 / #13. |

Bevy has no per-cascade switch for its own meshes: `NotShadowCaster` is all or nothing. The soldiers have per-cascade control only because they queue their own items into the shadow phase (render_units_shadow.rs).

Bevy's split for 3 cascades to 280 m: 0 to 40, 32 to 106, 85 to 280 m (the game logs it at startup), texels 3.3, 8.7 and 23.1 cm. Against 2 cascades to 110 m the second cascade shrinks from 110 to 106 m, so soldier shadows keep their sharpness. Receive levels are unchanged, L0 to L2 (`[true, true, true, false]` in the log): L3 starts past 400 m, outside either range.

## What a third cascade costs

Three separate costs, none of them from soldiers:

1. The view itself: Bevy prepares, culls, batches and opens a pass for every cascade every frame, whatever it holds. About 0.2 ms of render thread per cascade, from 0145 (three views about 0.3 to 0.7 ms) and 0147 (two cascades 0.3 to 0.6 ms in the CPU-bound views). It shows in the CPU-bound views (40 m, 900 m); in the flank view the GPU is the limit and it mostly hides.
2. Lookups: with 110 m, pixels past 110 m skip the shadow fetch (`fetch_directional_shadow` returns 1 past the last cascade). With 280 m the ground and the L2 soldiers between 106 and 280 m do the filtered fetch. For scale, the fetch for the near soldiers cost 0.4 ms in the flank view (0147).
3. Terrain from 85 to 280 m drawn into cascade 2. Terrain and the other scene meshes were about 0.15 ms of the 0.37 ms shadow pass in the flank view with 2 cascades (0147).

The trees themselves are cheap: 100 oaks are 200 meshes to cull per cascade and at most about 0.3 M triangles into one more map.

## Measurements

- Sandbox scene, no soldiers: shadow pass 0.11 ms with 3 cascades to 280 m, 0.08 to 0.11 ms with 2 to 110 m. Render-bound, not a battle number.
- Gota's 200k battles, fps counter, flank area, several runs each: 114 to 115 fps with 2 cascades to 110 m, 110 to 113 with 3 to 280 m. A few fps, about 0.2 to 0.4 ms, possibly run-to-run variance. For reference, the sun shadows branch read 110 to 115 with shadows on (0147).
- Confirmed 2026-09-29 by a controlled A/B against main (devlog 0163): 111.3 to 108.1 fps at the west end and 107.6 to 105.7 at the east end, 0.17 to 0.27 ms, the shadow pass 0.30 to 0.40 ms. Gota's reading was right and not variance.

## Decision

3 cascades to 280 m is the default on feat/vegetation, e593c33 ("Default to three shadow cascades out to 280 m"): mid-distance trees look much better, and the few fps are recoverable (plan below). `FL_SHADOW_CASCADES=2 FL_SHADOW_DIST=110` restores the old range. The comments in `setup_world` and on `ShadowReceiveLevels` now state the 280 m range. Build, strict clippy and a 15 s Sandbox scene run are clean (tmp/runs/merge-main-vegetation/scene-3casc.log).

## Perf plan

In order, each gated on Gota's fps counter in the 200k flank view (docs: fps-is-the-gate, flank angle recipe).

### 1. Terrain out of the shadow maps

`NotShadowCaster` on the terrain chunks (terrain.rs, a graphics-branch file). The ground keeps receiving every soldier and tree shadow; only its own casting goes. Removes cost 3 above and the terrain's share in cascades 0 and 1 today. Before it goes in: look at the grassland, classic and river maps with the terrain casting and without, at this sun, including the river banks and terraces. If a later sun retune wants hill relief in shadow (0145 left "hills show relief at 250 m" to the sun retune), terrain casting comes back with it. Small change, could ride on feat/vegetation. Not approved yet.

### 2. A static sun map for things that never move (next branch)

Redrawing trees that never move into a fresh map every frame is waste. Past 100 m no soldier casts, so every shadow out there comes from static things, and a map drawn once can serve it.

- One orthographic depth map from the sun over the whole field (1024 x 768 m: 16 x 12 chunks of 32 cells of 2 m). 4096 x 4096 gives 25 cm texels, as sharp as cascade 2 (23.1 cm). 32 MB at 16 bit, 64 MB at 32 bit float.
- Drawn once at battle start, again on a map switch or a vegetation respawn. Holds trees now, the bridge and buildings later. No soldiers, no terrain.
- Past the cascades, the terrain shader, the unit shader and the trees read it; inside, they read the cascades as now. Blend across the seam so no line shows.
- Cascades go back to 2 to 110 m: the 0.2 ms view and the tree redraw go away, and the cost no longer grows with the number of trees. The per-frame cost is one fetch per pixel past 110 m.

Pieces, hardest first:

1. The trees' own canopy past 110 m. Bevy's `StandardMaterial` has no clean hook for a second shadow map: either a custom tree material, or canopy shading baked into the asset (item 3) so trees need no lookup at all.
2. The one-time draw: our own depth-only pass over the tree meshes with the foliage alpha test, like the soldiers' shadow pipeline in render_units_shadow.rs. Bevy has no "draw this shadow map once".
3. The reads: assets/shaders/terrain.wgsl (Astra's file) and src/shaders/unit_instancing.wgsl (ask-first). L3 soldiers stay without a receive path.
4. Checks: every map by eye, the seam in motion, the flank-view fps gate.

Several sessions. Its own branch off main after vegetation merges, before or with the 0.3.0 authored battlefield (docs/plans/018), where maps with many trees arrive.

### 3. Canopy shading in the tree asset (Astra)

Bake the canopy's self-occlusion into the oak (vertex colour or texture), so a tree reads the same inside and outside any shadow map. Removes the pale pop at the 280 m edge now and at the 110 m edge once item 2 lands, and lets item 2 skip the custom tree material. 0156 already lists cavity/occlusion around the forks as a trunk lever; this is the same idea for the crown.

### 4. A Shadows setting with range, if wanted before item 2

Shadows: Off / Near (2 cascades to 110 m) / Far (3 to 280 m), replacing the on/off toggle under Video. `apply_shadows` (settings.rs) already switches the light live; it would also write the light's `CascadeShadowConfig`, which Bevy turns into cascades every frame. One setting rather than a slider per number: count and distance belong together (the count keeps near shadows sharp as the distance grows). The env vars keep overriding for A/B runs. Moot once item 2 lands.

### 5. Forest maps

Tens of thousands of trees would hurt the camera view before shadows: per-entity culling, LOD selection (`select_tree_level` walks every tree each frame) and extraction. Such maps need vegetation drawn through a GPU-driven path like the soldiers'. Item 2 covers their shadows.

## Open

- Terrain out of the shadow maps waits on Gota's go and the look check.
- Whether Astra's accepted foliage colours still hold under self-shadowing (the canopies read darker in debug13 than in the shadowless captures they were chosen from). Handoff: work/handoffs/069-tree-shadows-for-astra-2026-09-29.md.
