# 0147: Sun shadows against the fps gate, measured in Gota's view
Written by Claude Fable 5.1.

Date: 2026-09-27. Branch `feat/sun-shadows` at 00d68e4, five commits on main 66eb416, not merged. Follows devlog 0145 (what was built) and work/notes/021-shadows-perf-followups-2026-09-27.md (the review's follow-up list). No code change in this entry: it records where the branch stands against the gate, why, and what to do.

## The gate

Gota's counter in his 200k battle, shadows off against on: 135 to 145 against 125 to 135 in the mid view, 128 and up against 100 to 110 zoomed in from a flank with the army in the background, about 160 either way far out.

## Why devlog 0145 missed it

The three measurement views of 0145 (40 m, 280 m, 900 m, locked, looking down) are CPU-bound: the GPU finishes in 1.5 to 4 ms of a 5.5 to 6.5 ms frame, so the GPU shadow pass hid behind the CPU and only the CPU per-view cost showed. Gota's zoomed view is a different regime: camera at the left end of the line, 15 m out, pitch 0.35, looking along the front, 165k to 180k soldiers drawn (127k at L3, 37k at L2). There the unit pass alone is 6 to 7.5 ms and the GPU runs at 96 to 98%, the long pole. Every GPU millisecond shows on the counter, and the shadow pass grows with the fight as the near zone fills with casters: 0.6 ms when the lines meet, 1.2 ms two minutes in.

Recipe for that view, `FL_TEST_FRONT=1 FL_UNITS=100000 FL_CAM_LOCK=1 FL_CAM_X=-470 FL_CAM_Z=0 FL_CAM_YAW=-1.5708 FL_CAM_PITCH=0.35 FL_CAM_DIST=15`, AI on, 120 s, the last 60 s averaged. The test front is deterministic enough that runs match each other to within a few soldiers drawn at the same second, so windows line up between runs.

## Results

Flank view, live 120 s, mean of the last 60 s. `main` is the branch's base 66eb416, built in a scratch worktree with the shared target dir and run from a copy.

| Flank view, 200k | fps | frame | GPU unit pass | GPU shadow pass | render thread p50 | main thread update+post p50 |
|---|---|---|---|---|---|---|
| main | 135 | 7.4 ms | 6.13 ms | 0 | 4.9 ms | 2.6 ms |
| branch, `FL_SHADOWS=0` | 117 | 8.6 ms | 7.21 ms | 0 | 5.6 ms | 2.8 ms |
| branch, `FL_UNIT_SHADOWS=0` (scene casts, soldiers do not) | 112 | 8.9 ms | 7.40 ms | 0.15 ms | 5.6 ms | 3.0 ms |
| branch, coarse casters (`FL_SHADOW_FILTER_TEXELS=5`) | 109 | 9.2 ms | 7.42 ms | 0.37 ms | 5.9 ms | 2.9 ms |
| branch as shipped | 103 | 9.7 ms | 7.37 ms | 1.25 ms | 6.6 ms | 3.0 ms |

Locked 26 s views, mean of the last 12 s:

| | 40 m fps / frame / unit pass | 900 m fps / frame / unit pass |
|---|---|---|
| main | 186 / 5.4 ms / 1.05 ms | 188 / 5.3 ms / 1.53 ms |
| branch, `FL_SHADOWS=0` | 162 / 6.2 / 1.33 | 173 / 5.8 / 1.78 |
| branch, coarse casters | 162 / 6.2 / 1.27, shadow pass 0.26 | 171 / 5.9 / 1.79 |
| branch as shipped | 156 / 6.4 / 1.28, shadow pass 0.57 | 161 / 6.2 / 1.80 |

A load spike hit the box during the 900 m pair (GPU clock stayed at 1.92 GHz); treat those two rows as rough.

Earlier runs the same day, all 200k: at 15 m looking down (1k to 6k soldiers drawn) shadows cost 0.7 ms per frame, 201 to 177 fps, half of it on the render thread. At 40 m over a 170 s live fight the shadow pass went 0.16, 0.6, 1.1 to 1.25 ms as the casters grew from 400 to 13k (cascade 0 about 3.4k at L1, cascade 1 about 8k at L2); corpses were 1.5k of them, not the driver.

## Reading

Relative to main, the branch with coarse casters costs 1.75 ms per frame in the flank view (135 to 109 fps). It splits three ways:

1. **1.15 ms: the lighting commit (63f2b48), not the shadows.** `FL_SHADOWS=0` is not main: that commit moved the unit lighting from the vertex to the fragment so the shadow map could be sampled per pixel, and the vertex output grew from 10 floats (colour, team, atlas uv) to 16 (world position, world normal and the death term added). The flank view pushes 60M vertices through the pose shader per frame (127k L3 soldiers at 52 triangles, 37k L2 at 250, non-indexed pull), so every extra interpolant is paid 60M times, while the per-pixel math runs about 8M times. Consistent with the other views: the unit pass is 0.25 to 0.3 ms over main at 40 m and 900 m with shadows off, in proportion to their vertex counts. This is a hypothesis backed by the numbers, and the fix below is its test.
2. **0.6 ms: coarse shadows.** 0.37 ms of shadow pass (soldiers at L2 into cascade 0, L3 into cascade 1, scene 0.15 of it) plus 0.2 ms for the per-pixel fetch in the unit and terrain shaders. As shipped it is 1.2 ms: the pass alone is 1.25 with L1 into cascade 0 and L2 into cascade 1. Crops of the foreground knights at both settings (tmp/runs/sun-shadows/crop-flank-ft-pair.png) look the same to me: the Gaussian filter's footprint is 5 texels wide, so silhouette detail finer than that is smeared anyway. Devlog 0145's "2 texels" was a guess at the blur; 5 is the footprint.
3. **About 0.2 ms of render-thread CPU with shadows off**, present in every view: the shadow pass timer finishes two extra command buffers per frame (each Core3d system that touches the encoder ends one), and the shadow view bind group is rebuilt every frame. Both are mine and both are gateable.

The CPU per-cascade cost of 0145 (about 0.7 ms per frame from three extra views) is real but shows only in views where the CPU is the pole. In the flank view the GPU is, so the levers there are GPU levers.

Cascades against casters, since the two got mixed up in discussion: a cascade is one shadow map covering a band of distance from the camera (0 to 40, 32 to 106, 85 to 280 m), and everything in the band is drawn into it and reads its shadow from it. The caster level is which mesh a soldier is drawn with into a map, and it applies to every soldier who casts, not only the near ones. Soldiers stop casting past 100 m, so the far map holds banner and terrain shadows only. In the overlaps a pixel reads both maps and blends them. Picture, drawn by Claude Opus 5.5 and checked against these numbers: work/notes/vis/021-shadow-cascades-how.png.

The elephant in the flank view is on main already: the unit pass at 6.1 ms without any shadow, 60M vertex invocations of the full pose shader for soldiers that are three pixels tall. A cheap L3 vertex path (position, yaw, death topple, no gait or arms) is the next perf item for this view, plan 017 territory, not this branch.

## Plan

In order, Gota's counter after each:

1. Merge main (PR #11, no overlapping files), rebuild, regenerate the four gate baselines (the map heights changed).
2. Vertex output back to 10 floats, keeping the per-pixel fetch: colour+flash and team+death as two packed u32 (8 bits per channel, the atlas has no more), lambert per vertex split into ambient and direct so the shadow scales only the direct term (2 scalars), the shadow normal bias applied in the vertex shader from the cascade the vertex's view depth picks, so the fragment carries only the biased world position and the view depth (4), atlas uv (2). Measure the flank view against main; the shadows-off row should land on main's numbers.

   Per-vertex lambert is exact on these meshes, not an approximation: they are flat-shaded, every vertex of a face carries the face normal, so the lit value is constant across the face either way. What it does not carry is what Gota plans next: materials, armour that shines like M2TW's. A highlight needs the normal and the view direction per pixel, and a gloss or normal map in the atlas needs them per texel, so the near levels will need world position and normal back in the vertex output when that lands. That is not a conflict with this fix, because the cost lives in the far levels: of the 60M vertices in the flank view, L2 and L3 are about 47M and cover a few pixels each, L0 and L1 are about 13M and cover most of the screen. The pipeline is already specialized per bucket (`PullPipelineKey` carries the bucket), so the material pass adds a shader def for L0 and L1 only: wide output, per-pixel normal, view vector and specular there; L2 and L3 keep the packed 10-float path with per-vertex lambert and the per-pixel shadow fetch, where a highlight on a three-pixel soldier could never show. The shadow normal bias stays per vertex in both: Bevy applies it with the geometric normal, not the mapped one.
3. Coarse casters as the default (filter footprint 5). 2 cascades to 110 m as the default, `FL_SHADOW_CASCADES` and `FL_SHADOW_DIST` to bring the third back for distant trees. `NotShadowCaster` on the four bar cubes and the selection ring of every regiment. Timer and shadow bind group only when the sun has shadow maps.
4. His counter. Expected in the flank view: about 125 fps against main's 135, from 109 today.

## After the fixes

Same day, later. Main merged (cf18e4f, PR #11's textured terrain included), then five commits: 86c945f the vertex output back to eleven floats with an atlas and nine without, with the sun in a small uniform of its own because Bevy binds its lights to the fragment stage only; 172cb55 casters one level coarser (the filter footprint is five texels); ab228d6 two cascades to 110 m with `FL_SHADOW_CASCADES` and `FL_SHADOW_DIST`; 70c0be6 the banner bars and marker no longer cast; 14e91af the pass timer and the shadow view bind group only with shadows on. Then a Shadows toggle in the video settings, wired to the sun's `shadow_maps_enabled`, with `FL_SHADOWS=0` still over it. Fingerprints: the merged branch equals main on all four scenarios, baselines regenerated as `main6` (the map heights changed with PR #11).

`main` below is 6166f8b, current main with the textured terrain, which by itself costs about 0.8 ms of opaque pass in the flank view: main went from 135 fps there with the old ground to 122 with the new. Same recipe as above, quiet box, GPU clock 1.8 to 1.9 GHz.

| Flank view, 200k | fps | frame | GPU unit pass | GPU shadow pass | GPU opaque pass | render thread p50 |
|---|---|---|---|---|---|---|
| main | 122 | 8.2 ms | 6.23 ms | 0 | 0.88 ms | 5.2 ms |
| branch, `FL_SHADOWS=0` | 118 | 8.5 ms | 6.41 ms | 0 | 0.93 ms | 5.0 ms |
| branch, shadows on | 109 | 9.2 ms | 6.82 ms | 0.32 ms | 0.98 ms | 5.5 ms |

| | 40 m fps / frame / unit pass | 900 m fps / frame / unit pass |
|---|---|---|
| main | 182 / 5.5 ms / 1.06 ms | 179 / 5.6 ms / 1.53 ms |
| branch, `FL_SHADOWS=0` | 176 / 5.7 / 1.07 | 172 / 5.8 / 1.57 |
| branch, shadows on | 172 / 5.8 / 1.16, shadow pass 0.19 | 163 / 6.2 / 1.71 |

Rerun once more after the focus cap was understood (83be076 keeps the loop continuous when the window is unfocused, so the branch's fps reads frames generated from now on; main's binary still caps): main 123, shadows on 109, shadows off 120, within 2 fps of the table.

Reading: with shadows off the branch is now 0.2 to 0.3 ms from main in every view, down from 0.5 to 1.15; the vertex output was the cost, as the hypothesis said (unit pass 7.21 to 6.41 ms in the flank view). Shadows on cost 0.8 ms in the flank view, 122 to 109 fps: 0.32 of shadow pass, 0.4 of per-pixel fetch in the unit pass, a little in the terrain. In the CPU-bound views the two cascades cost 0.3 to 0.6 ms. The rest of the way to main in the flank view is the unit pass on main itself, 6.2 ms for 60M vertices, the L3 far path.

The look is unchanged with shadows off (crops of the same moment against the shipped build), and the close-up with shadows on keeps the shadows at the feet with no acne.

## Gota's counter, same day

His battle, 200k, the branch at 61c817d: from the far right flank looking along the line toward the far left, shadows on 110 fps (resources/shadow/debug4-shadow.png), off 120. A moderate zoom-out 125 to 135, the whole battlefield 150. At 110 the camera still moves smoothly, no felt lag. Main at the same angle 130 (resources/shadow/debug2-main.png), from the far left looking right about 120 (resources/shadow/debug3-main.png): the angle is worth 10 fps by itself, and his readings line up with the flank runs above. Merge gate met on his word; the coarser casters and the toggle click are his to judge in play.

Shadows off is still 0.2 to 0.3 ms behind main. All four levels of every kind are imported and textured, so every soldier takes the atlas path, and there the vertex output is one float wider than main's (11 against 10: the sun's share, the packed team and flash, the shadow position, the uv), plus two pack operations per vertex. At 60M vertices in the flank view one float is 240 MB of interpolant traffic a frame, which is about what the gap measures. It is not free, and the fix is a per-level shader variant: with the cascades ending at 110 m no L2 or L3 soldier is ever inside one (L2 starts past 100 m at this window size), so those buckets can drop the shadow position they never use and carry 8 floats, under main's 10. L0 and L1 keep the 11 and the fetch. About an hour, and it belongs with the L3 far-path item.

## The receive variant (9bbacf7)

The per-level variant landed on the branch. A level compiles the shadow receive path only when its nearest switch distance, jitter counted, lies within the last cascade's far bound, decided once a frame from the level bands: with the cascades at 110 m that is L0, L1 and L2 (L2 can start at 91 m), and L3, which starts past 400 m, drops the shadow position. With shadows off no level receives and every soldier carries eight floats against main's ten.

Flank view, same recipe, quiet box: shadows off 126 fps, 7.95 ms, unit pass 6.01 ms, against main's 123, 8.16, 6.20. Shadows on 111, 9.00 ms, unit pass 6.68 plus 0.30 of shadow pass, against 109 before the variant. So the branch with shadows off is now faster than main in the view that was the gate, by the two floats L3 no longer carries, and the interpolant-width reading held a second time.

The 40 m and 900 m rows took three batches: the first two came back with the main thread 0.7 ms slower than in every earlier run of those views (update+post 3.5 to 3.6 ms against 2.7 to 2.9) while the GPU timers matched to the hundredth, the load average climbing to 7 with the game accounting for about 5: desktop use during the runs, which the CPU-bound views feel and the GPU-bound flank view does not. The third batch, desktop idle, main thread back at 2.7 to 2.8:

| | 40 m fps / frame / unit pass | 900 m fps / frame / unit pass |
|---|---|---|
| main | 182 / 5.5 ms / 1.06 ms | 179 / 5.6 ms / 1.53 ms |
| branch, `FL_SHADOWS=0` | 181 / 5.5 / 1.02 | 174 / 5.8 / 1.49 |
| branch, shadows on | 179 / 5.6 / 1.16, shadow pass 0.18 | 176 / 5.7 / 1.49 |

With shadows off the branch now sits on main's numbers in every view, and ahead of it in the flank view. Shadows on cost 1 ms of GPU in the flank view (0.3 of pass, the rest the per-pixel fetch on L0 to L2), and 0.1 to 0.3 ms elsewhere; at 900 m the unit pass fell from 1.71 to 1.49 because every soldier there is L3 and no longer carries the shadow path.

## Method notes

- A run whose main thread waits far longer for the render thread than the render thread takes (the `extract+wait render` leg at 6 to 11 ms against a 3 ms render thread, fps exactly 60) is a capped run: the game window lost focus and Bevy's default winit settings drop an unfocused window to 60 updates a second. It happens when Gota uses the desktop during a run. Every fps and frame number of such a run is void; the GPU pass timers still hold. Six runs today were discarded that way before the cause was found, and Gota had to point out that nobody asked him to keep the window focused. Rule from now on: say when a batch starts and how long it runs, and read frames generated for routine numbers, his counter at final checks. From 83be076 the branch's loop stays continuous when unfocused.

- `FL_SHADOWS=0` on this branch measures "shadows off", not "main". For a main comparison build the base commit: `git worktree add --detach <scratch>/flanks-base 66eb416`, `CARGO_TARGET_DIR=<tree>/target cargo build --profile opt-dev` there (18 s, deps shared), copy the binary out, restore the branch binary from a copy. The scratch worktree is still registered; `git worktree remove` clears it.
- The periodic log already splits a frame into GPU pass timers and CPU legs; no Tracy was needed to place the cost.
- Runs: tmp/runs/sun-shadows/flank-live-*.log, flank-fl_*.log, cmp-*.log, long-40m-*.log, z15*.log.
