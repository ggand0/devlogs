# 0247: occlusion culling of soldiers

Written by Claude Opus 5.5. Branch `feat/occlusion-culling` off main e567f3e, 2026-10-13, built from plan 025 (docs/plans/025-occlusion-culling.md). Merged as PR #31 on 2026-10-14 (main ab80162).

## The point

The unit pass drew every soldier inside the frustum, hidden or not. The GPU's depth test only saved the fragments of a hidden man; his vertex shader runs were paid in full. In the low flank view most of the army stands behind the first few ranks. This branch tests each soldier against last frame's depth before he is queued, and does not draw the hidden ones.

## What was built

- **The pyramid** (`render_units_occlusion.rs`, `shaders/unit_depth_pyramid.wgsl`). After Bevy's opaque pass (soldiers, terrain, trees) a compute pass takes the view's depth buffer to a pyramid of farthest depths. Level 0 holds one texel per 2x2 pixels, the farthest of their 4x MSAA samples, and each later level halves the one before. The pyramid persists into the next frame, with the camera matrix and viewport it was drawn with. It costs 0.12 ms at 2560x1360.
  - Plan 025 named Bevy's own builder. Bevy's pyramid rounds the screen down to a power of two and samples it, so its texels do not line up with pixels. Ours has an exact mapping: texel x of level k covers pixels [x, x + 1) * 2^(k+1), and the last texel of each level takes the odd row or column the halving drops, so it covers everything to the screen's edge. A unit test (`levels_cover_every_pixel`) walks every pixel through every level for four screen sizes.
- **The test** (`unit_build.wgsl` `occluded`). For a soldier inside the frustum, his box's eight corners go through last frame's matrix. The box is hidden when every pyramid texel under its screen rectangle holds a depth nearer than the box's nearest corner. The read picks the finest level where the rectangle spans at most 8 texels and loads all of them (at most 81), so a far man's rectangle does not spill into the sky the way a 2x2 read at a coarse level does. A box partly off last frame's screen, or reaching past the near plane, is never hidden.
- **What a hidden soldier keeps.** He takes the frustum-culled path for the draw only. He keeps updating his level (so he comes back at the right one), his shadow casts, his ring draws. He takes no pose slot, so the pose pass shrinks with the draw.
- **The box** (`BOUNDS`), per kind, one for the standing and one for the dying and the fallen, in his facing frame from his record position: forward, back, sideways, up, down.
  - Plan 025 asked for a unit test that poses every clip frame on the CPU. Only the GPU poses soldiers: there is no CPU skinning or part chain to run. The vertex shader does the measuring instead. Under `FL_OCCLUDE_CHECK=1` every drawn corner writes how far it reached from its soldier into a buffer (atomic max per kind, state and direction), read back and printed against the boxes. That measures the real pose code, clips and code pose, at every level.
  - Three battles in the low flank view (40k for four minutes with the clips, the same with `FL_CLIPS=0`, 8k for five and a half minutes) gave the reach; the boxes are that plus 15 cm. The spearman's spear reaches 3 m ahead; a fallen spearman's spear goes 3.4 m into the ground. A crouch lowers the whole figure, so the standing reach down passes the feet by about 0.4 m. The man-at-arms plays the knight's clips and takes the knight's box.
- **The camera cut.** The main world compares the RTS camera with last frame's: a focus or distance step over 15% of the distance, a yaw or pitch step over 0.1 rad, or a map rebuild skips the test for a frame. The fastest pan (Shift) covers 9% of the distance in a 30 fps frame. A resize remakes the pyramid and skips a frame too.
- **The probe** (`render_units_gpu.rs` `OcclusionProbe`). In the mid-air views the test hides nothing, and the first batch showed the pyramid costing 0.15 ms of frame time there for nothing. The readback now carries a word saying whether the test ran; on such a frame the main world adds up the vertex runs it saved (hidden soldiers times their level's welded vertex count, a lower bound). Below 3M runs (the pyramid's 0.13 ms at 44 ps a run, devlog 0174) the pyramid is built only every 16th frame and the test runs on the frame after; a probe that hides more turns it back on every frame. Count mode and the checks build it every frame.
- **Modes**, read once from `FL_OCCLUDE`: 0 off, 1 cull (the default since 8039d9c), 2 count (the test runs and counts, everything draws). `FL_GPU_CHECK` turns cull into count. The periodic log prints `unit occlusion (<mode>): hidden per level [...]` under the fps line.
- **A latent bug on main**: `selection_rings::init_ring_pipeline` read `MeshPipeline` at RenderStartup with no ordering against the system that inserts it. Adding one RenderStartup system reordered the schedule and the game panicked on the missing resource. Fixed in its own commit (d2f4757) by ordering it after `MeshPipelineSystems`, as `init_custom_pipeline` already was.

Commits: d2f4757 (ring pipeline order), 0d87dd4 (the test, off by default), aff8050 (the probe, the CPU path, map rebuild as a cut, the coverage test), 8039d9c (on by default), 5e3a2a1 and c5c670e (comment fixes from the text check and the final review on 2026-10-14).

## Where the build left plan 025, and why

- **Its own pyramid instead of Bevy's.** Bevy's builder rounds the screen down to a power of two and samples it, so its texels do not line up with pixels and a conservative read has to allow for the mismatch. Ours maps exactly (texel x of level k covers pixels [x, x + 1) * 2^(k+1), the last texel of a level takes the odd row or column), and the unit test `levels_cover_every_pixel` walks every pixel through every level for four screen sizes.
- **The box sizes measured in the vertex shader instead of a CPU unit test.** The plan asked for a test that poses every clip frame on the CPU, but only the GPU poses soldiers: no CPU skinning or part chain exists. `FL_OCCLUDE_CHECK=1` records how far every drawn corner reached from its soldier; over three long battles (clips on and off) that gave the reach, and the boxes are the reach plus 15 cm.
- **A same-frame check instead of a pixel diff between two runs.** Two runs never pose alike because the idle sway and the cheer read the clock, even frozen in deployment. Instead, count mode with `FL_OCCLUDE_CHECK=1` draws the soldiers the test would hide in flat magenta, and the magenta pixels in screenshots are counted: 0 with the camera still and orbiting (`FL_CAM_ORBIT`, added for this). The control that marks every tested soldier gives 2.5M magenta pixels, so the mark and the counter work.
- **The pyramid every 16th frame while it hides little.** Built every frame, it cost about 0.15 ms of frame time in the views from above for nothing. It is now built every 16th frame until a probe finds enough hidden soldiers (3M vertex runs), which removed that loss.
- **The branch name.** `feat/occlusion-culling`, as Gota asked; the plan had said `perf/occlusion-culling`.

## The checks

### Is the test conservative: the magenta check

A pixel diff between an `FL_OCCLUDE=0` run and an `=1` run cannot be exact here: the idle sway and the cheer read the clock, so two runs never pose alike, even frozen in deployment. The check runs in one frame instead. In count mode with `FL_OCCLUDE_CHECK=1`, a soldier the test calls hidden is still drawn, flat magenta, in his place in the depth order. Any magenta left in the final image is a hidden soldier who shows. `work/scripts/perf/magenta.py` counts those pixels in a screenshot (`work/scripts/shots.sh`, which grabs the window with ImageMagick). `FL_OCCLUDE_CHECK=2` marks every tested soldier, which shows the mark and the counter work: 2.5 million magenta pixels.

| Run (low flank view) | Frames | Hidden soldiers | Magenta pixels |
|---|---|---|---|
| 200k battle, camera still, 7 fps under the check (large steps per frame) | 2 | about 150k | 0 |
| 200k battle, `FL_CAM_ORBIT=0.6` (0.02 to 0.05 rad a frame) | 8 | up to 155k | 0 |
| 40k, `FL_CAM_SWEEP=-20` | 7 | 0 (every frame a cut) | 0 |
| 40k at x = -120, camera still | 1 | 0 | 0 |

So the one-frame lag the plan worried about does not show at these speeds: the boxes are wider than the soldiers, and an occluder must move by that margin before the soldier behind it shows. `FL_CAM_ORBIT=<rad/s>` is new for this (camera.rs, the locked camera only). The sweep zooms faster than the cut threshold every frame, so the test stays off through it. The look in play, panning and zooming by hand, is still Gota's to check.

### What hides the soldiers

Count mode, 200k, the hidden soldiers with the soldiers in the pyramid and without them. `FL_UNITS_FIRST=0` draws the soldiers in Bevy's transparent phase, after the pyramid pass, so the pyramid then holds only the ground and the trees:

| View | Hidden by the ground and trees | Hidden by those and the soldiers | In view (L2, L3) |
|---|---|---|---|
| Low flank (x = -470) | 110k (68k L2, 42k L3) | 164k (78k, 86k) | 86k, 85k |
| Its mirror (x = 470) | 4.5k | 28k | 86k, 85k |

So in the low flank view most of the hidden soldiers stand behind a rise in the ground: the battle is in the grassland's lowland and the camera is 5 m up at one end of it. Near soldiers hide another 54k. In the mirror view the ground in front is open, and the soldiers do most of the hiding. In the 40k scene at x = -120 nothing is hidden: seen from 5 m up, every rank's head stands a little higher on screen than the one in front, as devlog 0173 worked out, and the boxes reach a metre above the heads. L0 and L1 are never hidden in any view: the near soldiers have nothing in front of them.

### The bounds

`FL_OCCLUDE_CHECK=1` over the three battles above printed no reach outside the final boxes; the reach per kind and state is in the `BOUNDS` comment. Coverage is only what those battles played: the archers' melee reach showed up in the third battle only (forward 1.50 m against 0.51 m in the first two), which is why the margin is 15 cm and not 10. A pose not seen there (the victory cheer did not happen in those runs; the hop is 0.1 m) can still leave the box; the check is there to find it.

### The gain

The perf rule (memory `measuring-perf`): the AI battle (`FL_TEST_FRONT=1`), 2560x1360, camera locked, 120 s, the mean of the last 60 s, against main e567f3e built from `git archive` (`tmp/runs/perf4/bin/flanks-main`). The runs were interleaved main, branch, main, branch. Load average 4 to 14 during the runs: the docker containers and the desktop were idle (97% CPU idle before the batch), and most of the load average is the game's own threads. GPU clock 1800 to 1905 MHz a minute into each run. Script: `work/scripts/perf/occl-bench.sh <bin> <view> <units per side> [ENV]`, summary `work/scripts/perf/occl_summary.py`, logs in `tmp/runs/perf4/logs/b2-*` (the adaptive build) and `main-*`/`occl-*` (the first batch, pyramid every frame).

Second batch, the build as committed (aff8050 plus the default):

| View | Soldiers | main fps (ms) | branch fps (ms) | Frame | Unit draw | Vertex runs |
|---|---|---|---|---|---|---|
| Low flank | 200k | 161.1 (6.21) | 204.8 (4.89) | -1.32 ms | 4.65 to 2.43 ms | 70.5 to 33.7 M |
| Low flank | 300k | 135.0 (7.40) | 173.6 (5.76) | -1.64 ms | 5.82 to 2.61 ms | 91.3 to 38.1 M |
| Low flank mirror | 200k | 153.5 (6.52) | 161.0 (6.21) | -0.31 ms | 4.95 to 4.42 ms | 70.1 to 62.4 M |
| Low flank mirror | 300k | 129.7 (7.71) | 140.8 (7.11) | -0.60 ms | 6.03 to 5.29 ms | 90.9 to 78.9 M |
| Mid-air | 200k | 159.9 (6.25) | 159.3 (6.28) | +0.03 ms | 4.17 to 4.18 ms | 54.6 to 54.6 M |
| Mid-air | 300k | 132.2 (7.57) | 131.0 (7.64) | +0.07 ms | 5.53 to 5.59 ms | 77.3 to 77.3 M |
| Mid-air mirror | 200k | 152.9 (6.55) | 153.6 (6.51) | -0.04 ms | 4.33 to 4.31 ms | 54.8 to 54.8 M |
| Mid-air mirror | 300k | 129.1 (7.75) | 128.5 (7.78) | +0.03 ms | 5.69 to 5.75 ms | 77.4 to 77.4 M |

The pose pass shrinks with the draw (0.43 to 0.14 ms in the low flank view at 200k). The pyramid pass is 0.12 to 0.14 ms when it runs.

First batch, the pyramid every frame (0d87dd4 with `FL_OCCLUDE=1`): the same gains in the flank views, and in the mid-air views +0.14 to +0.18 ms, which is what the probe removed. The branch with `FL_OCCLUDE=0` matched main (161.6 against 159.9 fps at 200k, 136.2 against 137.0 at 300k).

One pass time moved without a frame time to match: `sun_shadows` reads about 0.15 ms more with the test on in every view, the mid-air views included, where the pyramid is built only every 16th frame. That span is two raw encoder timestamps around the shadow pass (render_units_shadow.rs `ShadowPassTimer`), not a pass timestamp, and the frame times in the mid-air views are equal to main's. Not measured further.

### Gota's measurement (2026-10-14)

Gota's own battle and views on the branch build, the fps counter read by eye over two runs each (the second run in parentheses):

| View | main | branch |
|---|---|---|
| Ground, one side | 160 to 165 after a while (163 to 165) | 168 to 173 (168 to 173) |
| Mid-air, one side | 146 to 150 (137 to 139) | 150 to 151 (146 to 148) |
| Ground, the other side | 155 to 158 | 161 to 163 (158 to 163) |
| Mid-air, the other side | 130 to 134 | 141 to 143 (136 to 138) |

Gota's reading: the far right flank ground view gained, because the ground slopes there, which matches the ground hiding most of the hidden soldiers above. The branch reads as fast as or faster than main in every one of Gota's views. The mid-air gains of 4 to 10 fps there are larger than the benchmark's mid-air views showed (equal within 0.07 ms); main's mid-air view varied by 10 fps between Gota's two runs, so they are within that spread, and the hidden count in those views was not logged.

### Other paths

30 s runs, no errors: `FL_MSAA=1` (the single-sample pyramid pipeline, 0.07 ms), `FL_GPU_SYNC=0` (the CPU path, no test, no pyramid), `FL_GPU_CHECK=1` (forces count mode; 1302 of 6000 frames differed by one soldier, and main differs the same way in the same scene, 1286 of 6000), and no flags at all. Strict clippy and the 45 unit tests pass.

## Open

- The look in play: pan, orbit and zoom fast in the low flank view and watch the silhouettes of the near soldiers for a soldier behind them appearing a frame late. The magenta check says there is none at the speeds above.
- Tighter boxes would hide more. The standing knight's box is 3.1 m wide because his swings reach 1.4 m to the side; a soldier who is not attacking reaches much less. A box per state (idle, fighting) is the next step if the mirror or mid-air views need more.
- The shadow casters are not tested against the sun's view (out of scope in the plan).
- `FL_UNITS_FIRST=0` keeps working: the pyramid then holds the ground and the trees only.
