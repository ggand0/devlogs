# 0170: 60 fps at 1440p in a 200k battle: the unit pass at L2

Written on Windows 10 in the NTFS tree at `D:\ggando\gamedev\flanks`, for copying into the Linux devlogs. 2026-10-01. RTX 3090, Ryzen 9 3900X, primary monitor 2560x1440 (work area 2560x1400), a second monitor in portrait. Windows clone at `C:\Users\gotag\projects\flanks`.

## The question

With the display at 2560x1440 and the game window maximized (not fullscreen), a 200k battle reads 60 to 65 fps on the stats line. At a 1600x900 window the same box reads 125 to 130 (devlog 0168). What is the bottleneck, and what brings a maximized 1440p window to 100 to 120 fps or more?

The maximized client area is about 2560x1360: the work area is 1400 rows and the title bar takes the rest. That is 2.42 times the pixels of 1600x900 and 1.51 times the rows.

## Answer, from the code and the logs on disk

The audit below ran nothing. It reads the code, the handoffs and the 0.2.0 release test log (`tmp\release-0.2.0\test-run.log` in the Windows clone). Measurements that test it follow in their own section.

The bottleneck is the unit draw (`main_transparent_pass_3d`), and it is vertex work, not pixels. Three things stack.

1. The level switch distances grow with the window's height. At 1440p the whole default view lands on L2.
2. An L2 soldier costs 4.7 times an L3 soldier in vertex shader runs.
3. Every one of those runs computes the soldier's whole pose again.

### 1. Level switch distances follow the window height

A level is picked by the soldier's height on screen in pixels: 28 px and taller is L0, 12 to 28 px is L1, 3 to 12 px is L2, under 3 px is L3 (`LOD_PX_DEFAULT`, src/render_units.rs:132). The pixels per metre come from the physical viewport height (`build_frame_params`, src/render_units_gpu.rs:309). With Bevy's 45 degree vertical field of view that is `rows / 0.8284` px per metre at 1 m.

Distance at which a 1.0 m soldier (man-at-arms, spearman, archer) changes level. The knight is 1.1 m, so his distances are 10% longer. Each soldier also has a ±10% jitter.

| Window rows | L0 to L1 | L1 to L2 | L2 to L3 |
|---|---|---|---|
| 900 (1600x900) | 39 m | 91 m | 362 m |
| 1080 (1920x1080) | 47 m | 109 m | 435 m |
| 1360 (maximized at 1440p) | 58 m | 137 m | 547 m |
| 2160 (4K borderless) | 93 m | 217 m | 869 m |

The default camera sits 280 m from its focus at a pitch of 0.9 rad, so 219 m above the ground. The bottom edge of the screen looks at ground 228 m away, the centre 280 m, the top edge 451 m. At 900 rows the top part of the view, past 362 m, is L3. At 1360 rows every soldier in the default view is L2 or finer.

This is the rule working as designed: a soldier as many pixels tall gets the same level at any resolution. The cost is that a taller window puts more soldiers on the finer levels.

### 2. What each level costs per soldier

Triangle counts from the release test log:

| Kind | L0 | L1 | L2 | L3 |
|---|---|---|---|---|
| Knight | 2768 | 678 | 250 | 52 |
| Man-at-arms | 2998 | 674 | 248 | 48 |
| Spearman | 2992 | 664 | 250 | 56 |
| Archer | 2987 | 680 | 250 | 56 |

The pulled draw expands each level's index list into a corner list (`PullMesh::from_mesh`, src/render_units_gpu.rs:588) and draws without indices. So the vertex shader runs three times per triangle: about 8,900 runs per L0 soldier, 2,000 per L1, 750 per L2 and 160 per L3. L2 costs 4.7 times L3.

The asset spec (docs/plans/009-unit-asset-spec.md) calls L2 "the level seen most in play" and sets it at 150 to 250 triangles for 3 to 12 px. A soldier 4 px tall covers fewer than ten pixels, so at the far end of the band nearly every L2 triangle is smaller than a pixel. The GPU still sets up each one, and with 4x MSAA it tests four samples on each.

### 3. The pose runs once per corner

`unit_vertex` (src/shaders/unit_instancing.wgsl:675) takes a corner and poses it from the soldier's record: gait, stride and hip drop for every corner, then per part the leg (`leg_pose`, with `stance_ankle`, `heel_lift` and `leg_ik`: acos, atan2, sqrt, a Hermite swing), the weapon arm (`arm_pose` or the jointed chain with its attack tables), the bow rig, the shield arm, then the shoulder turn, lean, stagger, death topple and yaw. Only the corner's rest position and part change between the 750 corners of one L2 soldier. The leg and arm angles come out the same for all of them.

### Cost per corner

The release test run (20k battle, 1920x1080 window, default camera, one periodic log line):

- 5,642 soldiers at L2 and 5,185 at L3, about 5.1M corners: unit pass 0.40 ms.
- That is about 80 ps per corner.
- Terrain, vegetation and water (`main_opaque_pass_3d`): 0.24 ms.
- Sun shadows: 0.10 ms. Unit build compute: 0.03 ms.

The GPU clock was not logged. A light view like this one lets the 3090 clock down (devlog 0078), so 80 ps may overstate the full-clock cost. Devlog 0147 read about 6 ms for 60M corners in the flank view at 200k, 100 ps per corner, and that view held full clock.

### What it adds up to at 200k

With 150k to 180k soldiers on screen, all on L2, the unit pass draws 110M to 135M corners. At 50 to 80 ps each that is about 6 to 11 ms for the unit pass alone. The frame budget for 120 fps is 8.3 ms for everything.

At 900 rows the far third of the same view is L3, at about a fifth of the corners. The step from 7.8 ms (125 to 130 fps) to 16 ms (60 to 65 fps) fits this.

### What else grows with the window, and why it is not the main cost

- Pixels grow 2.42 times. The opaque pass (textured terrain with Bevy's PBR lighting and shadow lookups, vegetation, water) was 0.24 ms at 1920x1080. At 1360 rows that is about 0.4 ms.
- The camera carries Bevy's default 4x MSAA (src/camera.rs:57 sets none). It adds coverage work on every small triangle. Worth an A/B, but after the geometry.
- The shadow cascades do not depend on the window. Units cast with L2 into cascade 0 and L3 into cascades 1 and 2 at any size.
- Vegetation picks its detail by logical viewport height (src/vegetation.rs:1190), so it grows with the window too. It lands in the opaque pass.

## Plan, in order

### 1. Pose once per soldier, not once per corner

The build compute pass (`unit_build.wgsl`) already runs one thread per soldier and computes his animation state. It would also need the four kinds' rigs and shot tables. It writes each soldier's part transforms to a buffer: each leg's thigh, shin and foot, the weapon arm's shoulder, elbow, wrist and weapon, the shield arm, the bow rig's joints, the body's turn, lean, stagger, death and yaw. The vertex shader reads its part's transform and applies it. The knee, elbow and torso blend bands stay a mix of two transforms by the corner's rest position.

The work per corner drops to a few loads and one or two rotations. Only soldiers on L0 to L2 need the buffer, since L3 is all body. It helps every zoom, the shadow casters too, and it is what 1M soldiers needs. The pose code moves as it is, so the A/B is an `FL_` switch and screenshots with forced levels, as in item 2 (devlog 0082).

### 2. Right-size the 3 to 6 px band

The switch to L3 sits at 3 px because L3 is all body (spec 009), so its legs cannot walk. An L3 that keeps two legs and the weapon as parts, at 60 to 80 triangles, could take over from about 6 px and still march. At 1440p that takes roughly half of the default view off L2. It changes spec 009 and is model work. `FL_LOD_PX` lets the switch be judged by eye first.

### 3. Later: sprites for the far band

Medieval II draws distant soldiers as sprites. A sprite sheet rendered at startup from L1 through the same pose shader (facing directions, gait phases, a few pose states, the team colour through the mask) would draw each far soldier as one quad: 4 to 6 corners instead of 156 to 750. That is the way to 1M. It risks visible popping at the switch and baked lighting, so it comes after 1 and 2.

### Smaller, to measure after item 1

- An indexed draw for L1 and L2: one instance per soldier, the level's own index buffer, the soldier found by instance index. It gives back the vertex reuse the corner list throws away. Devlog 0078 measured 1.5 times fewer shader runs on L0 indexed, against about 8 ns per instance. It pays only while the vertex shader is heavy, so it is decided after item 1.
- MSAA 2x or off, once the geometry is fixed. Spears will shimmer without it.

### Not doing

- Capping the level bands at 900 rows, or rendering below the native resolution. Both are the 1600x900 window again in another form.

### Expected

Items 1 and 2 together should bring the 1440p unit pass to about the 900p cost or below: about 110 to 130 fps. That is an estimate, not a measurement. Past about 130 the CPU frame with a sim tick is the next limit.

## The test that confirms or breaks it

At a 2560x1360 window, `FL_LOD_PX=42.3,18.1,4.53` (the defaults times 1360/900) puts every level switch at its 1600x900 distance. Only the pixel count then differs from a 1600x900 run. If fps comes back near 110 to 125, the vertex load is the cause. If it stays near 65, it is the pixels and the plan above is wrong.

## Measurements (2026-10-01, Windows)

Branch `perf/unit-pose-per-soldier` off main 53f0610 in the Windows clone. opt-dev builds. Benchmark script `tmp\bench.ps1` in the Windows clone (not committed): the 200k test front (`FL_TEST_FRONT=1 FL_UNITS=100000`, both armies ordered into contact) with the camera locked at the -x end of the front, low, looking along the line (`FL_CAM_X=-470 FL_CAM_Z=0 FL_CAM_DIST=15 FL_CAM_PITCH=0.35 FL_CAM_YAW=-1.5708`, the devlog 0147 view). This is the heaviest kind of view: about 184k soldiers drawn, near ones at L0. The overhead default view is not a fair test and is not used. Each run averages battle blocks 15 to 34 (30 to 70 s into the fight), shadows on, GPU clock logged beside the game log (P0, 1.86 to 1.92 GHz in every run). `FL_WINDOW=WxH` (commit 84107af) opens the window at an exact physical size.

| Run | fps | Unit pass | Opaque pass | Shadows | Soldiers per level L0/L1/L2/L3 |
|---|---|---|---|---|---|
| 1600x900 | 92.8 | 8.19 ms | 1.02 ms | 0.39 ms | 1970 / 6627 / 55883 / 118997 |
| 2560x1360 | 58.4 | 13.29 ms | 2.08 ms | 0.51 ms | 4366 / 13114 / 87820 / 78494 |
| 2560x1360, LOD switches at their 900-row distances (`FL_LOD_PX=42.3,18.1,4.53`) | 82.1 | 8.45 ms | 2.09 ms | 0.41 ms | 2043 / 6889 / 55933 / 118933 |
| 2560x1360, `FL_MSAA=1` | 61.1 | 13.10 ms | 1.53 ms | 0.51 ms | same as 2560x1360 |
| 2560x1360, pose skipped in the vertex shader (probe, not kept) | 77.7 | 9.25 ms | 2.01 ms | 0.39 ms | same |
| 2560x1360, `FL_LOD_PX=28,12,6` | 73.8 | 9.72 ms | 2.10 ms | 0.50 ms | 4365 / 13114 / 28251 / 138069 |
| 2560x1360, `FL_LOD_PX=40,18,8` | 98.4 | 6.87 ms | 1.87 ms | 0.25 ms | 2277 / 6742 / 22107 / 152682 |

What they say:

- The hypothesis holds. With the switch distances put back to 1600x900's, the 1440p unit pass is 8.45 ms against 8.19 at 900p. The level bands cost about 4.8 ms of the 1440p frame. The rest of the gap to 900p is pixels, mostly the opaque pass (terrain, vegetation, water): 2.09 against 1.02 ms.
- The unit pass follows the corner count: about 89 ps per corner at 900p (91.8M corners) and 92 ps at 1440p (144M corners). In the flank view at 1440p that is L0 27%, L1 18%, L2 46%, L3 9% of the corners.
- MSAA is small: 0.7 ms across both passes.
- The pose is about 4 ms (30%) of the unit pass at 1440p: skipping it in the vertex shader leaves 9.25 ms. That was the ceiling for item 1.
- The other 9.25 ms is geometry volume: 48M triangles a frame, about 14 per pixel, even with a near-empty vertex shader. Coarser levels move fps the most (98 fps at 40/18/8 px), but they change the look, so the thresholds and the far level models are Gota's call.
- The per-level meshes (tools: a Python GLB reader in the session scratchpad) keep their parts grouped, so warps rarely span more than two parts. L1 to L3 are flat shaded with almost no shared vertices (1.1 corners per vertex), so an indexed draw gains nothing there. L0 has 2.2 to 2.5 corners per vertex, so an indexed L0 draw would run its vertex shader about half as often.

## Item 1 built: the pose pass (commit f39a032)

- `shaders/unit_pose.wgsl` is a new shader module: the rig, the pose helpers moved over unchanged, `pose_begin` and the `put_*` functions (everything about a soldier's pose that does not depend on the corner), and `place` (the per corner part: rest position through the part's joints, the knee, ankle, elbow and torso blend bands, the lean, stagger, fall and facing).
- The build pass gives each drawn soldier (camera or shadow caster, living or fallen) a pose slot in his kind's region and puts the slot in the list entries. The pose pass (`shaders/unit_pose_pass.wgsl`) runs one indirect dispatch per kind with the kind's rig bound and writes 17 slots of four floats per soldier, 45 for a kind with a bow rig. One pose buffer per kind, sized for the kind's cap plus its corpse ring: about 100 MB at 200k.
- The pulled draw reads the pose. The instanced CPU path and `FL_POSE_PASS=0` pose per corner through the same code, filling only the slots the corner's part reads.
- Checked against the original per-corner code with a temporary harness (not committed): the pulled vertex shader also ran the old `unit_vertex` and painted any corner more than 1 mm or any normal more than 0.001 off in magenta. Zero magenta pixels in 17 screenshots: the 200k flank clash, the archery test mid-volley, the charge test, the saved 200k battle with the AI, and the per-corner path. At zero tolerance only float rounding shows (spear shafts, a few parts), under 2 mm.
- Build, strict clippy and the 22 tests clean.

| Run | fps | Unit pass | Pose pass | Shadows |
|---|---|---|---|---|
| 2560x1360, before | 58.4 | 13.29 ms | none | 0.51 ms |
| 2560x1360, pose pass | 74.6 | 9.46 ms | 0.23 ms | 0.39 ms |
| 2560x1360, `FL_POSE_PASS=0` | 56.7 | 13.73 ms | none | 0.53 ms |
| 1600x900, before | 92.8 | 8.19 ms | none | 0.39 ms |
| 1600x900, pose pass | 113.3 | 5.95 ms | 0.24 ms | 0.30 ms |

The pose pass recovers 3.6 of the 4 ms the probe put as the ceiling. The per-corner path is 3% slower than before the split, which only the A/B and the CPU fallback use.

## Three more look-neutral changes (commits 54a2fcf, b335e22, 9cfe068)

### Indexed L0 (54a2fcf)

A level whose triangles share at least 1.5 corners per vertex keeps its own vertices and index list next to the expanded corners. Its camera draw is `draw_indexed_indirect`, one instance per soldier (the soldier from `instance_index`), with the arguments written by `finalize` into a small `indexed_args` buffer. Only the four L0 levels qualify (2.2 to 2.5). Shadow casters keep the expanded corners. 1440p flank: unit pass 9.46 to 9.16 ms, 74.6 to 76.1 fps.

### The soldiers draw before the terrain (b335e22)

The units were opaque but queued in Bevy's transparent phase, so they drew after the ground and every ground pixel behind them was shaded for nothing. `src/render_units_phase.rs` adds a sorted phase of their own (`Units3d`), registered through `SortedRenderPhasePlugin::<Units3d, MeshPipeline>` like Bevy's own transparent phase, with its pass in `Core3dSystems::MainPass` before `main_opaque_pass_3d`. Bevy's attachments clear on first use, so the unit pass clears and the opaque pass loads. Buckets draw level-major, near levels first. `indexmap` becomes a direct dependency only because the phase trait names its type (same version bevy builds). `FL_UNITS_FIRST=0` queues into the transparent phase for A/B.

1440p flank: opaque pass 2.13 to 0.49 ms, 74.7 to 84.9 fps, unit pass unchanged. 900p: 123.4 fps.

### Tipsify on the indexed levels (9cfe068)

The exported L0 order shades about 2.0 vertices per triangle under a FIFO cache model (16 or 32 entries), against a floor of 1.23 to 1.37 distinct vertices per triangle. Tipsify (Sander, Nehab and Barczak 2007) with a 16-vertex cache reaches the floor for all four models (Python model of the GLBs, then a Rust port with a unit test). In game it did more than the model predicted: 1440p flank unit pass 9.39 to 8.49 ms, 84.9 to 92.9 fps. 900p: 5.80 to 5.32 ms, 127.6 fps.

## Final numbers (one binary, 9cfe068, no builds during the runs)

| Run | fps | Unit pass | Opaque | Shadows | Pose pass |
|---|---|---|---|---|---|
| Flank clash, 2560x1360, at start (53f0610) | 58.4 | 13.29 ms | 2.08 ms | 0.51 ms | none |
| Flank clash, 2560x1360 | 92.9 | 8.49 ms | 0.41 ms | 0.39 ms | 0.24 ms |
| Flank clash, 1600x900, at start | 92.8 | 8.19 ms | 1.02 ms | 0.39 ms | none |
| Flank clash, 1600x900 | 127.6 | 5.32 ms | 0.27 ms | 0.30 ms | 0.27 ms |
| Saved 200k battle, AI on, flank camera, 2560x1360 | 108.3 | 6.87 ms | 0.65 ms | 0.31 ms | 0.25 ms |
| Same, `FL_POSE_PASS=0 FL_UNITS_FIRST=0` | 75.2 | 9.58 ms | 2.10 ms | 0.42 ms | none |
| Flank clash, 2560x1360, `FL_LOD_PX=36,16,6` | 124.1 | 5.42 ms | 0.42 ms | 0.32 ms | 0.27 ms |
| Flank clash, 2560x1360, `FL_LOD_PX=40,18,8` | 138.7 | 4.40 ms | 0.40 ms | 0.23 ms | 0.27 ms |

The saved battle is Gota's own last setup (`%APPDATA%\flanks\last_setup.yaml`, copied to the clone's `tmp\bench-200k-setup.yaml`): 100k a side, 1000 per regiment, grassland, AI on, 38% men-at-arms, 27% knights, 23% spearmen, 12% archers, run with `FL_SETUP`, `FL_AUTOSTART=1`, `FL_DEPLOY=0` and the same flank camera. The old-paths row still has the indexed L0 and its reorder (no switch), so the true before for that battle is somewhat lower. Gota's own report for a battle like it, maximized at 1440p, was 60 to 65 fps.

Screenshots of every run sit next to their logs in the clone's `tmp\runs\bench\`, taken at the same battle moment (block 25), so the threshold runs can be compared by eye with the default.

### Gota's own check

Gota ran the branch build on Windows with the game window maximized on the 2560x1440 display, in a 200k battle, from the spot he always uses to check performance by hand. The stats line read 108 to 110 fps. The same check read 60 to 65 fps before this work (the question at the top of this devlog).

## What is left

- The unit pass is now about 59 ps per corner at 1440p and follows the triangle count. More speed without changing the look is small from here.
- The level thresholds are the big lever and a look decision: 36/16/6 gives 124 fps and 40/18/8 gives 139 fps in the heaviest view at 1440p. Raising the L2 to L3 switch past 3 px freezes the legs of soldiers 3 to 6 or 8 px tall, since L3 is all body. An L3 that keeps two legs and the weapon as parts (60 to 80 tris, spec 009) would allow it without that cost.
- Sprites for the far band (Medieval II's answer) stay the path to 1M.
- The CPU: frames that carry a sim tick take 7.8 ms (900p) to 11.3 ms (1440p flank) at p50 against 5.7 to 10.2 without. Above about 120 fps the tick frames are the ceiling, and that is sim work.
- Memory: the pose buffers take 17 slots of 16 bytes per soldier, 45 with a bow rig, sized per kind for its cap plus its corpse ring: about 100 MB at 200k.

## The failures, plainly

- The first benchmark used the default overhead camera. Gota pointed out that it draws the fewest soldiers and so hides the cost. Every number above uses the flank view along the front.
- Two PowerShell variables collided with parameters of the same name in another case (`$blocks` and `-Blocks`, `$shots` and `-Shots`). The second left a game I had started running. I closed it.
- A `cargo test` compile ran while a threshold benchmark was running, against the rule in CLAUDE.md. Those two runs were discarded and rerun on the final binary with nothing else running.
- Python edits passed through a bash heredoc lost their backslashes twice. Edit scripts went into files after that.
- Screenshots are of the whole primary screen, so the Claude app shows around the game window in some of them. They stay in the clone's `tmp`.
- The Windows clock went back about four hours during the session (game logs at 17:39 UTC, then 13:56 UTC). Cargo decides freshness by file times, so it treated edits made after the jump as already built. One "the shaders still load" launch ran the old binary. The two edited files got timestamps past the last build, a real rebuild followed, and the checks were redone.

## Before merging (PR #19)

PR #19 (https://github.com/ggand0/flanks/pull/19), `perf/unit-pose-per-soldier` at b74ebda. Main has not moved since the branch point 53f0610, and GitHub reports the PR mergeable with no conflicts. The repository has no CI workflows.

Two commits after the numbers above:

- f219e2f "Fix the pose module's comments": the ten em dashes the comments brought from the unit shader became commas, colons and full stops. Comments only.
- b74ebda "Require ten storage buffers for the GPU unit path": the pose pass and the indexed draw raised the build pass from 8 to 10 storage buffers, but `GpuUnitRenderPlugin::finish` still started the GPU path at 8. A GPU allowing 8 or 9 per stage would have drawn no soldiers. It now keeps the CPU path. Only weak or old GPUs are affected.

Final checks on b74ebda:

- Text: no footer, no tool or model names, no "owner" or person names, no hype words in the commit messages or the added comments. Three commit bodies run to four sentences against the one to three of the writing guidelines. Left as is (a force-push to fix).
- Build, strict clippy and the 23 tests clean after the real rebuild. Launches on the GPU path and on the CPU path (`FL_GPU_SYNC=0`) load every shader without errors.
- Arrows ride the soldiers' phase through the per-vertex pose path, which the pose check did not cover. In the archery test the arrows show stuck in the ground among the target block.
- Selection rings draw under the new phase (`FL_SELECT_ALL=1` check). Hover rings follow the mouse, so runs are not comparable on them.

Not checked, skipped for now by Gota: the water on the river map. Water is `AlphaMode::Blend`, so it stays in Bevy's transparent phase. Before this branch the soldiers were in that phase too, sorted against the water by the distance of their mesh centres, so which drew first depended on the camera position. Now the soldiers always draw first and the water always blends over whatever part of a soldier is below the surface. A wading soldier on `FL_MAP=river` should show the water tint over his legs from every camera angle, where before it depended on the angle. Screenshots of both phases at the river (`FL_MAP=river`, camera at x 115, z -12) are in the clone's `tmp\runs\quick\*final-river*.png` and have not been looked at.

Not run: Linux, macOS, the fat-LTO release build.

## Index row for devlogs/README.md

| 0170 | 2026-10-01 | [60 fps at 1440p in a 200k battle: the unit pass](0170-unit-pass-audit-at-1440p.md) | At a maximized 1440p window the level switch distances grow 1.5 times and most of the army lands on L2, whose 250 triangles were each shaded three times with the whole pose worked out per corner. Measured in the 200k flank clash: 58 fps at 1440p, 93 at 900p. Four look-neutral commits on perf/unit-pose-per-soldier: a pose pass that poses each drawn soldier once, an indexed L0 draw, the soldiers drawn before the terrain, Tipsify on L0. Now 93 fps in the flank clash at 1440p, 128 at 900p, 108 in Gota's own saved battle at 1440p, and 108 to 110 in Gota's manual check from his usual spot, maximized at 1440p (60 to 65 before). Coarser level thresholds reach 124 to 139 and are a look decision. PR #19. Water on the river map now always draws over wading soldiers, not checked by eye |
