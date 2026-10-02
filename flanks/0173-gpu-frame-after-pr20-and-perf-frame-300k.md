# 0173: What the GPU spends after PR #20, and branch perf/frame-300k

Written on Windows 10 in the NTFS tree at `D:\ggando\gamedev\flanks`, for copying into the Linux devlogs. 2026-10-02, 03:20 to 06:00, worked unattended while Gota slept. RTX 3090, Ryzen 9 3900X, display 2560x1440. Windows clone `C:\Users\gotag\projects\flanks`, main at 5a78afd (PR #21 merged on top of PR #20). Follows devlog 0172.

## The question

After PR #20 Gota reads 148 to 155 fps in the 200k clash, 140 to 144 along the front, and 128 to 135 in a heavier mid-air view. The goal stays 200k to 300k at 144 fps daily, 500k at 120 or more later. Gota asked for an unattended night: find the new bottlenecks, pick the few most worth doing, and start a branch on them, measuring in the views that draw the most soldiers.

## Gota's check, in his words (2026-10-02 morning)

Gota played perf/frame-300k (88b3b29, the same code as d9d58c7, which only rewraps a comment) against main the next morning:

> Morning, this is great. From my usual far right flank to far left looking spot on the ground, now it's 158-163 (142-147 on main). This is like two units away from the actual far right flank edge, but I've been using this spot for a while so I keep this spot. When you benchmark you shouldn't cheat and your spot is probably better.
> I'd say this view is kind of solved these days.
>
> a mid-air spot that is two mouse scroll down away from there using my Razor mouse reads about 138-141 with occasional dips to 136-137 (128-131 for example on main), though this spot can vary when I manually run. Different from those spots, but when I intentionally moved around the camera to look for the worst spot, I found this spot > D:\ggando\gamedev\flanks\tmp\flanks_perf2.png and flanks_perf3.png
> This is a similar mid-air view but the bottom edge is literally the right most edge of far right flank. This probably renders the most amount of soldiers, and on this branch it's 125-129 (120-126 on main) in the worst or something. Worth noting

The worst spot of `flanks_perf2.png` and `flanks_perf3.png` (mid-air, the bottom edge of the frame on the outermost edge of the far right flank) is not fitted to `FL_CAM_*` values yet; it is the candidate for a third benchmark view.

## The two benchmark views

- Low flank view (devlog 0147): `FL_CAM_X=-470 FL_CAM_Z=0 FL_CAM_DIST=15 FL_CAM_PITCH=0.35 FL_CAM_YAW=-1.5708`, `tmp\bench.ps1 -View flank`.
- Mid-air view, fitted to Gota's `D:\ggando\gamedev\flanks\tmp\flanks_perf1.png` by eye: `FL_CAM_X=-400 FL_CAM_Z=0 FL_CAM_DIST=140 FL_CAM_PITCH=0.28 FL_CAM_YAW=-1.5708`, now `tmp\bench.ps1 -View midair`. Candidates tried (screenshots in `tmp\runs\midair\`): distance 90 to 150, pitch 0.28 to 0.43, x -470 to -400, the saved battle and the scripted front. The scripted front clash at 45 s matches the screenshot's composition (blue left, orange right, the front up the middle to the horizon, the blob to the bottom edge, the horizon about a fifth down) at x -400, distance 140, pitch 0.28. The saved battle has gaps between regiments that the screenshot does not.
- Both views from the other end of the front, looking back: `-View flankmirror` and `-View midairmirror` (x +470 and +400, yaw +1.5708).

In the front clash at 30 to 50 s the mid-air view draws about 192k soldiers, levels [14, 13338, 87935, 90658], against 186k, [4346, 13190, 89050, 79503] in the low view: no L0, the same L1 and L2, more L3.

All runs below: `FL_WINDOW=2560x1360` (the maximized 2K window), the scripted front clash (100k a side) unless noted, battle blocks 15 to 24 of `tmp\bench.ps1` (30 to 50 s), nothing else running.

## Where the frame goes at 200k

Baseline in this session (main 5a78afd, a probe build with no probe switched on):

| View | fps | frame | unit pass | opaque pass | unit shadows | GPU busy (nvidia-smi) |
|---|---|---|---|---|---|---|
| Low flank | 138.7 | 7.22 ms | 5.02 ms | 0.57 ms | 0.28 ms | 95% |
| Mid-air | 138.9 | 7.19 ms | 3.86 ms | 1.81 ms | 0.22 ms | 98% |

Both views are GPU-bound: nvidia-smi reports the GPU busy 95 to 98% of the time. The frame is about 0.9 ms longer than the sum of the timed GPU passes in every variant below; that is GPU work the overlay does not time (it is not Bevy's own shadow pass: with `FL_SHADOWS=0` the frame shrinks by the timed shadow costs and about 0.06 ms more). The main thread spends about 3.3 ms per frame (fixed loop 0.5, Update and PostUpdate 2.8) and waits the rest.

### The unit pass is bound by the triangles it sets up

Temporary probe switches (scratch build, not committed): a constant fragment, no raster (every corner moved to one point after the vertex shader ran, so the triangles are dropped as degenerate), no pose math (each corner placed at its rest position plus the soldier's), and skipping one level's draw.

| Unit pass, ms | Low flank | Mid-air |
|---|---|---|
| Base | 5.02 | 3.86 |
| Constant fragment | 4.19 | 3.00 |
| No raster | 3.53 | 2.62 |
| No pose math | 4.85 | 3.67 |
| No pose math, no raster | 3.53 | 2.60 |
| Without L0 | 3.44 (-1.58) | |
| Without L1 | 4.21 (-0.81) | 2.51 (-1.35) |
| Without L2 | 3.12 (-1.90) | 1.79 (-2.07) |
| Without L3 | 4.75 (-0.27) | 3.51 (-0.35) |

- The pose math is 0.17 to 0.19 ms. With the raster gone it is nothing: the vertex stage costs the same with or without it.
- What is left without raster, 3.5 ms (low) and 2.6 ms (mid-air), is the geometry front end. The pipeline statistics (now in the log, below) count 50.9M triangles in the low view and 37.4M in the mid-air view: about 70 to 76 ps per triangle, at the 1.87 GHz the card holds, about 7 triangles per clock, one per raster engine (GA102 has seven). Every level costs about its triangles times 75 to 90 ps, plus its fragments: L2's 22M triangles 1.9 to 2.1 ms, L0's 12.8M 1.6 ms (larger triangles, more fragments).
- Vertex reuse is what the model said: 72.7M vertex shader runs for 50.9M triangles, 1.42 per triangle in both views, Tipsify's estimate in devlog 0171. Nothing left there.
- Fragments: 9.70M (low) and 6.35M (mid-air) invocations. The shadow lookup is 0.42 to 0.45 ms of them, the atlas sample 0.25 to 0.27 ms (probes with the receive path or the atlas sample removed).

So the unit pass shrinks only with fewer triangles or fewer fragments. Fewer triangles is a look decision (below). Fewer fragments can be had without one (item 3).

### The terrain shader in the mid-air view

From above the ground fills most of the frame and Bevy's opaque pass, nearly all of it the terrain, costs 1.81 ms against 0.57 in the low view. The terrain shader was swapped at run time (it loads from `assets/`, no rebuild):

| Mid-air, opaque pass | ms | fps |
|---|---|---|
| As on main | 1.81, 1.79 | 138.3, 137.0 |
| Earth layer skipped where its weight is 0 | 1.10, 1.07 | 151.0, 148.8 |
| Flat colour, no textures, no lighting | 0.28 | 170.4 |
| Bevy's lighting, no textures | 0.32 | 168.6 |

Bevy's lighting is almost free; the textures are about 1.5 ms. The earth layer's six samples (three patches, colour and normal) were taken on every pixel, though on the grassland its weight (`field.a`, the authored soil mask, and the painted vertex green) is exactly 0 on most of the field. Item 1.

### The draw order of the soldiers

Within a bucket the soldiers draw in the order the build pass appended them, which follows the soldier index. The same scene seen from the other end of the front:

| | Low flank | Low, mirrored | Mid-air | Mid-air, mirrored |
|---|---|---|---|---|
| Unit fragments | 9.70M | 10.80M | 6.09M | 6.32M |
| Unit pass | 5.11 ms | 5.47 ms | 3.85 ms | 3.94 ms |

Same triangles, 11% more fragments shaded from one end: soldiers behind are shaded before the ones in front cover them. Item 3.

### 300k

`FL_UNITS=150000`, same views: 114.7 fps (low) and 113.4 (mid-air), the GPU busy 98%, unit pass 6.60 and 5.64 ms. The main thread spends about 4.2 ms of an 8.6 ms frame, and the tick job (grid 5.2 to 5.7, step 13 to 13.6, field 0.7 ms) about 19.5 ms of its 33 ms window. So up to 300k the GPU sets the frame rate; the CPU has room. 300k at 144 fps needs about 1.8 ms off the GPU frame.

## Ruled out, with the reason

- Occlusion culling of whole soldiers. With the camera above head height, a farther soldier's head always projects above a nearer one's, so almost no soldier is hidden in full, and a conservative depth pyramid test could not cull him.
- Index lists per view direction (each level drawn with only the triangles that can face the camera from where it is). To stay exact under the pose (torso twist, limb swings of 20 to 45 degrees or more) a triangle can be dropped only when its normal is within about 20 degrees of facing away: a few percent of the triangles.
- The atlas sample in the vertex shader for the levels with one atlas point per triangle: 53M vertex runs against 6.35M fragments.
- Thread placement of the sim: the frames are GPU-bound, the job's overlap is hidden behind the GPU.

## Look decisions, measured, for Gota

None of these is on the branch. Each changes what the far soldiers look like, so each is Gota's call.

| `FL_LOD_PX` (L0, L1, L2 minimum heights) | Low flank fps | Mid-air fps | Level split, low view |
|---|---|---|---|
| 28,12,3 (main) | 138.7 | 138.9 | 4346/13190/89050/79503 |
| 28,12,4 | 148.2 (+7%) | 149.1 (+7%) | 4346/13191/58788/109763 |
| 28,12,5 | 156.7 (+13%) | 157.0 (+13%) | 4345/13192/40688/127865 |
| 36,16,4 | 164.9 (+19%) | 159.3 (+15%) | 2773/8177/65375/109764 |

(Before the branch's items; they add on top.)

- L2 is 250 triangles for a soldier 3 to 12 pixels tall, 22M triangles a frame, the largest single cost. A level of about 120 triangles between L2 and L3, or the spec 009 L3 with legs and weapon, is model work.
- The far soldiers' shadow lookup: 0.42 to 0.45 ms. A single hardware-filtered tap for L2, or no shadow received past L1, would take most of it.

## Branch perf/frame-300k

Off main 5a78afd, on origin, PR #23. The three items in the order picked: the two that are exact first, then the one that changes only which hidden fragments get shaded.

### 1. The terrain's earth layer only where it shows (edb11b7)

`assets/shaders/terrain.wgsl`: the earth layer (`sample_ground` over `earth` and `earth_normal`, three patches, colour and normal) is sampled after the coverage, and only where `max(field.a, in.color.g) > 0` on the authored ground; the river maps keep it everywhere. Every later use of it is a `mix` at weight `field.a`, `in.color.g` or their maximum, and `mix(a, b, 0)` is `a`, so a pixel where both are 0 comes out the same.

Pixel check (`tmp\terrain-check-setup.yaml`, one 200-man regiment a side out of shot, the real battle view in deployment, mid-air camera, client area): 0 of 2.3M pixels differ, the dry patches and the far meadow corridor in frame. The comparison swaps the shader file at run time on the same binary.

Illustration: `work/notes/vis/terrain-earth-layer-skip.png` (drawn by `tmp\tools\vis_earth_layer.ps1`).

Measured with the shader swapped on the same binary, two rounds, mid-air view: opaque pass 1.81 and 1.79 ms to 1.10 and 1.07, 138.3 and 137.0 fps to 151.0 and 148.8.

### 2. The pipeline statistics in the periodic log (ebdd68a)

`overlay.rs` logs, per render pass, the vertex shader runs, triangles set up (`clipper_invocations`) and fragment shader runs Bevy's `RenderDiagnosticsPlugin` already records where the device has pipeline statistics queries (the 3090 on Vulkan has). Every number in the unit pass analysis above comes from them.

### 3. The near levels drawn near to far

Build pass changes (`unit_build.wgsl`, `render_units_gpu.rs`):

- `build` counts each drawn soldier or body into his bucket and one of 128 depth bins (log2 of the squared camera distance, 1 m to 4096 m, about 7% of distance per bin) and keeps his bin, his slot in it and his list entry in a new `draws` buffer (12 bytes per build thread, binding 11; the build layout now needs 11 storage buffers per stage).
- `finalize` turns each bucket's bin counts into the first slot of each bin.
- A new `scatter` dispatch writes each entry at its bin's first slot plus its slot.
- Only L0 and L1 use the bins (`ORDERED_LODS = 2`); L2 and L3 keep one bin, the order the build found them, which is what main draws.
- `FL_UNIT_ORDER=0` puts every soldier in one bin: the order main draws.

How it got there:

- First all four levels ordered. Two alternating rounds, `FL_UNIT_ORDER` 0 against 1 on one binary: low view unit pass 5.13/5.20 to 4.99/5.06 ms (fps +1.6%), mirrored low view 5.54/5.51 to 5.21/5.25 (+3.5%), mid-air 3.92/3.94 to 4.08/4.06 (-1.3%), mirrored mid-air 4.07/4.12 to 4.13/4.16. Unit fragments: low view -16%, mirrored low -25%, mid-air -1.4%, mirrored mid-air -2.5%.
- With the pose pass off (`FL_POSE_PASS=0`) the mid-air view lost nothing (4.83/4.86 against 4.85/4.86 ms), so the pose buffer's access order was the first suspect: the pose slots are handed out in build order, the draws read them in depth order. Handing them out in draw order (slots, `pose_src` and the caster appends moved into `scatter`, the caster arguments into a step after it) changed nothing: mid-air 3.85 against 4.02 ms. Reverted.
- Ordering level by level (a probe switch, `FL_UNIT_ORDER_LODS`), mid-air and mirrored low view, unit pass: none 3.87 / 5.54, L0 3.86 / 5.40, L0 and L1 3.86 / 5.29, L0 to L2 3.98 / 5.29, all 4.06 / 5.27. The gain is all in L0 and L1, whose soldiers are large on screen and hide the ones behind; the cost is all in L2 and L3. The likely reason: a far soldier is a few pixels, and in the build's order his neighbours are his neighbours in the regiment, writing the same 4x MSAA colour and depth tiles; in depth order a level's consecutive soldiers lie along a band across the screen, so each touches tiles the GPU no longer holds. Not measured directly.

The final version (d9d58c7), `FL_UNIT_ORDER` 0 against 1 on one binary, two alternating rounds:

| View | Unit pass off | Unit pass on | fps off | fps on |
|---|---|---|---|---|
| Low flank | 5.19, 5.26 ms | 4.98, 5.02 ms | 140.9, 139.8 | 143.2, 142.5 (+1.8%) |
| Low, mirrored | 5.51, 5.56 | 5.25, 5.29 | 130.0, 128.7 | 133.9, 133.4 (+3.5%) |
| Mid-air | 3.95, 3.92 | 3.88, 3.91 | 148.8, 147.9 | 150.4, 148.5 (+0.8%) |
| Mid-air, mirrored | 4.12, 4.17 | 4.00, 4.03 | 137.9, 137.7 | 140.1, 139.2 (+1.4%) |

Image check (`tmp\probes\order_imgcheck.ps1`, frozen deployment frames of the saved battle, both views, client area): order on against off differs in 834 and 1964 pixels, two runs with it off in 1131 and 2347 (the leg smoothers settle with the frame rate). Only exact depth ties could change, which the noise hides. Strict clippy and the 28 tests clean.

## Branch against main

`tmp\probes\ab_branch.ps1`: main (`flanks-probe.exe`, no probe switched on, main's terrain shader) against d9d58c7 (`flanks-branch.exe`), alternating, two rounds, front clash, blocks 15 to 24:

| View | main fps | branch fps | main unit pass | branch unit pass | main opaque | branch opaque |
|---|---|---|---|---|---|---|
| Low flank | 140.1, 137.6 | 143.8, 143.5 (+3.4%) | 5.04, 5.15 ms | 4.97, 4.98 ms | 0.53, 0.47 ms | 0.41, 0.38 ms |
| Low, mirrored | 128.3, 127.5 | 133.8, 133.0 (+4.3%) | 5.46, 5.52 | 5.21, 5.29 | 0.68, 0.69 | 0.59, 0.59 |
| Mid-air | 138.7, 138.1 | 150.2, 148.4 (+8.0%) | 3.86, 3.91 | 3.87, 3.89 | 1.81, 1.77 | 1.08, 1.05 |
| Mid-air, mirrored | 133.9, 134.7 | 139.7, 140.0 (+4.1%) | 4.05, 3.97 | 3.99, 4.01 | 1.81, 1.82 | 1.56, 1.56 |

The mirrored mid-air view has the dry eastern meadow, where the earth layer is in use, close below it, so the terrain item takes 0.25 ms there instead of 0.7. The unit pass gain in the low view reads 0.1 ms here against 0.2 ms in the same-binary A/B above; both within the noise of one session against another.

Still short of 144 fps in three of the four views at 200k, and about 114 at 300k before the branch. The rest is the look decisions and the untimed GPU time below.

## The frame rate audit

Gota asked (2026-10-02) whether the fps every perf PR quotes is computed right, and whether it is the number on the stats line. This is the most important check of the night: an optimization measured on a wrong metric is not an optimization. Traced through `overlay.rs` and bevy_diagnostic 0.19:

- The stats line (top left) and the periodic `fps:` log line both call `overlay::frame_rate`: the mean of `FrameTimeDiagnosticsPlugin::FRAME_TIME` over its history, and 1000 divided by that mean. The bench reads the same number the player sees.
- `FRAME_TIME` is `Time<Real>::delta` in ms, the wall time between consecutive main-app frames (`frame_time_diagnostics_plugin.rs`). With pipelined rendering the main app and the render thread advance one frame each in steady state, so this is the time per finished frame.
- The rate is frames over elapsed time, which is the right one. Bevy's own `FPS` diagnostic averages each frame's 1/dt, which reads high when frame times vary (a 5 ms and a 10 ms frame are 200 and 100 fps, averaging 150, while the true rate is 2 frames in 15 ms, 133). The game does not use it.
- The history is Bevy's `DEFAULT_MAX_HISTORY_LENGTH`, 120 frames (bevy_diagnostic `lib.rs`): about 0.8 s at 150 fps. The comment on `frame_rate` said about two seconds; 570b5c0 corrects the comment, the number was never wrong.
- `tmp\bench.ps1` averages the logged fps (whole numbers) of battle blocks 15 to 24. Each log line holds the last 120 frames of its two seconds, so the ten lines sample 8 s spread evenly over the 20 s window, with the frames that carry a sim tick in their share (about 24 of 120 at 150 fps). Unbiased; the rounding averages out.
- It counts the frames the game finishes with vsync off, as game counters do. Above the display's 144 Hz not every frame is shown whole.

Gota's rule from this, for every perf claim: never optimize on a wrong metric, and measure in the environment players use, with a metric that measures what players care about. His example: earlier perf work was tuned at 1600x900 on Linux, and the game then ran much worse on Windows in a maximized 2K window (devlog 0168). That work still counted, but the numbers that decide are a player's window size, a player's OS, the views players actually look at, and a rate that is frames over time.

## Merged

PR #23 merged into main on 2026-10-02 as 48b3e0c, after PR #22 (the Windows exe with the assets inside). Five commits: edb11b7, ebdd68a, d9d58c7, 570b5c0, and 7c32e57, which rewords the comment on `finalize`: each camera bucket's thread turns its depth bin counts into the first slot of each bin. Checked before the merge on the tree main plus the branch: strict clippy clean, the 29 tests pass (#22 added one), and the 16 branch runs of the A/B logged no shader or validation error. Gota put it in 0.2.1, not tagged yet: one Linux run and one macOS launch first.

## Next, in order of what they buy toward 300k at 144 fps

1. The level thresholds and the far meshes (Gota's look call, table above): 7 to 19% now. The single biggest cost is L2, 250 triangles for soldiers 3 to 12 pixels tall.
2. The far soldiers' shadow lookup (look call): 0.42 to 0.45 ms of the unit pass.
3. The untimed 0.9 ms per frame. On the branch, mid-air view: 0.92 ms at 2560x1360 (GPU 95% busy), 1.12 ms with MSAA off (90% busy), 1.48 ms at 1600x900 (79% busy, partly CPU-bound there). It does not shrink with the pixels or the samples, so it is not the MSAA resolve or the window's composition. At 2560x1360 about 0.35 ms of it is the GPU idle and about 0.55 ms fixed per-frame GPU work that no timer covers. A GPU capture (PIX or Nsight Graphics, neither installed; installing one is Gota's call) would name it.
4. The terrain's remaining texture work, about 0.75 ms in the mid-air view (1.1 ms opaque pass against 0.28 with a flat colour). From mid-air roughly a million ground pixels each take about 13 texture samples: two layers (pasture, and earth where it shows) of three patches each, colour and normal (normals within 220 m only), plus the coverage, plus nine more coverage samples in the eastern meadow band, with 8x anisotropic filtering at grazing angles. Heavy for RTS ground. Gota (2026-10-02): an optimization job for Claude, worth a branch. Candidates: one patch instead of three beyond some distance, where the patch blend cannot be resolved anyway; lower anisotropy far away; the meadow band's nine samples replaced by a mask computed once at load. A guess of 0.3 to 0.5 ms, not measured. None of them is exact, so each needs screenshots for Gota's eye.
5. 500k: the tick job at 300k takes 19.5 of its 33 ms window, so plan 013 (the pair pass, the band rebuild) stays the sim work for 500k.

## Tools

- `tmp\bench.ps1 -View midair|flankmirror|midairmirror`.
- `tmp\tools\pass_stats.py <logs>`: the unit pass's vertices, triangles and fragments averaged over battle blocks.
- `tmp\tools\chrome_frames.py`: per-span averages split by tick frames from a Bevy `trace_chrome` file. Written, not used: the frames turned out GPU-bound.
- `tmp\probes\apply_probes.py` applies the probe switches (`FL_PROBE=frag|noraster|simplevs|noatlas|noreceive`, `FL_PROBE_SKIP=<levels>`) for a scratch build and keeps the originals in `tmp\probes\orig\`; never committed. Their binaries are `target\opt-dev\flanks-probe.exe` and `flanks-probe2.exe`.
- `tmp\probes\terrain\*.wgsl`: main's terrain shader (`orig`), the branch's (`branch`) and the probe variants (`flat`, `notex`, `skipsoil`).
- `tmp\probes\*.ps1`: the night's run scripts (`probe1`, `probe2`, `terrain_probe`, `ab_order`, `ab_branch`, `order_imgcheck`).
- `tmp\terrain-check-setup.yaml`: one regiment a side, out of the mid-air shot, for pixel checks of the ground.

## Index row for devlogs/README.md

| 0173 | 2026-10-02 | [What the GPU spends after PR #20, and branch perf/frame-300k](0173-gpu-frame-after-pr20-and-perf-frame-300k.md) | Unattended night on Windows. Gota's mid-air view fitted (x -400, distance 140, pitch 0.28) and scripted with mirrored views from the other end. At 200k and 300k the frame is GPU-bound (95-98% busy); the unit pass costs about 75-90 ps per triangle set up whatever its shaders do, L2 the largest share; the terrain shader 1.8 ms from mid-air. Branch: the terrain's earth layer sampled only where it shows (bit-identical, -0.7 ms mid-air), pass statistics in the log, L0 and L1 drawn near to far through depth bins and a scatter step (-21 to -27% fragments in the low views; ordering L2 and L3 made them slower). Level thresholds and the far shadow lookup measured for Gota's look call. Merged as PR #23 |
