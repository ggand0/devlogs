# 0145: Sun shadows, soldiers included (plan 012 item 3)
Written by Claude Opus 5.5.

Date: 2026-09-26. Branch `feat/sun-shadows` off main 66eb416, in the main tree. Handoff: work/handoffs/HANDOFF-sun-shadows-2026-09-26.md. Not merged, waiting on Gota's play test.

## Commits

- 3cddd91 Turn on the sun's cascaded shadow maps. Three cascades, first to 40 m, out to 280 m, 2048 maps. `FL_SHADOWS=0` turns the maps off.
- 63f2b48 Light units per pixel from the scene's sun and its shadow. The unit shader's sun was a constant (0.45, 0.85, 0.3), 58 degrees up, while the scene light is 43 degrees up from another azimuth. It now reads the first directional light from the lights uniform, and lighting moved from the vertex to the fragment so the shadow map is sampled per pixel. The first mesh transform hack (`get_world_from_local(0u)`) is gone: positions go to clip space from world space.
- 5fc2508 Cast soldier shadows into the sun's cascades. Unit buckets join Bevy's `Shadow` phase as non-mesh items with a depth-only variant of the unit pipeline. The build pass moved before the early shadow pass.
- 58f804b Time the sun's shadow pass on the GPU. Two timestamp queries around the early shadow pass, logged as `render/sun_shadows/elapsed_gpu`. wgpu becomes a direct dependency (same 29.0.4, no features).
- 00d68e4 Cull soldier shadow casters per cascade on the GPU. See below.

## How soldiers cast

The shadow variant is the unit pipeline with no fragment stage, the shadow map's depth format, one sample, and groups 0 to 2 replaced: group 0 carries only the light view uniform at binding 0 and the globals at binding 11, the bindings `bevy_pbr::mesh_view_bindings` declares, so the unit shader's imports and `vertex_pull` entry work unchanged. Groups 1 and 2 are empty. Group 3 is the bucket's own. Bucket entities carry `NotShadowCaster` so Bevy's own shadow queue skips them, and our queue takes the items out and puts them back every frame because the binned phase is retained.

The first version drew the camera's L0 and L1 lists into every cascade. It worked and looked right, but cost 3.7 to 4.2 ms of shadow pass in the 40 m view at 200k at full clock, because every cascade reran the whole pose shader for every near soldier: about 60 M vertex invocations against 20 M in the main pass.

A cascade box test in the build pass made it worse (6.7 ms): Bevy sizes each cascade as the bounding sphere of its frustum slice for stability, so the boxes take in every near soldier behind and beside the camera.

What ships: the build pass decides per soldier where his shadow can fall. His cull sphere, swept away from the sun by the height of its top above the ground, gives a sphere on the ground. If that sphere is in the camera frustum, he goes on the caster list of every cascade that is sampled at the view depths it spans (Bevy's cascade bounds with the blend band). Off-screen soldiers whose shadows fall in view cast too, which removes the limitation the handoff expected. Records are written for those off-screen casters, level hysteresis stays with the camera.

Each cascade draws a kind with one level, the level the camera picks for a soldier as many pixels tall as his shadow can show detail: his height in shadow texels over the 2 texels Bevy's Gaussian 5x5 PCF blurs together (`render_units::shadow_level`). With the default camera: cascade 0 (3.3 cm texels) L1, cascade 1 (8.7 cm) L2, cascade 2 (23.1 cm) L3. Casters are still only soldiers the camera would draw at L0 or L1.

GPU side: 16 caster lists (4 cascades by 4 kinds) after the camera lists in the index buffer, 16 more counters and draw arguments, and the bucket table grows to five 256 byte sets, the camera's and one per cascade. Each pulled bucket gets one extra bind group per cascade binding that cascade's set, so the unchanged vertex shader reads the cascade's list for its kind.

## Measurements

200k, `work/scripts/gpu-pass-measure.sh`, GPU core clock 1.8 to 1.95 GHz, medians of the last 12 s:

| View | Shadow pass | Unit pass on / off | Opaque pass on / off | Render thread p50 on / off | fps on / off |
|---|---|---|---|---|---|
| 40 m | 0.59 ms | 1.34 / 1.42 ms | 0.09 / 0.06 ms | 3.2 / 2.8 ms | 156 / 161 |
| 280 m | 0.10 ms | 3.52 / 3.50 ms | 0.13 / 0.10 ms | 3.1 / 2.8 ms | 171 / 175 |
| 900 m | 0.02 ms | 1.80 / 1.78 ms | 0.19 / 0.18 ms | 3.3 / 3.0 ms | 158 / 177 |

- Added GPU time in the 40 m view: about 0.6 ms, under the 1.5 ms target. The per-pixel shadow lookup in the unit pass is inside run noise.
- `FL_UNIT_SHADOWS=0` (Bevy's meshes only): 0.14 to 0.3 ms of shadow pass.
- Full texel detail (`FL_SHADOW_FILTER_TEXELS=1`, L0 in cascade 0): 2.15 ms at the end of the 40 m run, 34 M vertices. Close-up crops of both look the same to me.
- Caster counts in the 40 m view once the lines meet: about 2,500 in cascade 0 and 6,300 in cascade 1, none in cascade 2 (it starts at 85 m, past where soldiers stop casting).
- The render thread gains about 0.3 ms with shadows on in every view, the same at 900 m where the shadow pass draws nearly nothing: Bevy's fixed cost per extra view (three cascade views to prepare, specialize, batch, preprocess and open passes for). The 900 m fps gap (158 against 177) is larger than that in these runs and fps swung 154 to 173 within the one run, so it needs Gota's counter in his battle before it means anything. `perf` is locked on this box (`perf_event_paranoid` 4), so no sampled profile.
- Fingerprint gate (`work/scripts/gate.sh`, prefix `sunsh`): dir, arch, pilewide and pile2 all equal to main5. Nothing reached the sim.
- `FL_GPU_CHECK=1`: 4,500 frames compared, 0 differed. The camera's buckets are unchanged.

## Look

- Soldiers stand on their shadows in the close-up, no acne on soldiers or ground, default biases kept.
- Shadowed ground goes almost black: Bevy's ambient is tiny next to an 8,000 lux sun, while the unit shader keeps about half its brightness in shadow (its own hemispheric ambient). The two disagree. Balancing the ambient is the grade, Astra's pass after merge.
- The hills do not shade themselves at the current 43 degree sun on this gentle ground, so "hills show relief at 250 m" waits on the sun retune.
- At 280 m the far half of the view is past the 280 m shadow distance; distant banners cast nothing.

## Known gaps

- The CPU path (`FL_GPU_SYNC=0`) still draws the camera's near buckets into every cascade: 3.7 ms shadow pass at 40k in the close-up. Porting the per-cascade lists to the CPU sweep is the fix if the fallback matters.
- Unverified: Bevy's shadow queue does not skip blended materials, so the water surface probably shades the river bed. I could not frame the river in a test scenario to check. If it shows, a `NotShadowCaster` on the water entity fixes it, in water.rs, Astra's file.

## Knobs

`FL_SHADOWS=0` all sun shadows off. `FL_UNIT_SHADOWS=0` soldiers do not cast. `FL_SHADOW_FILTER_TEXELS=n` shadow detail per filtered texel (default 2, 1 = full texel detail).
