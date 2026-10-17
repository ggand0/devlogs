# Hill map benchmark at 200k

Written by Claude Opus 5.5. 2026-10-17.

The 200k battle on Big Hill (feat/hill-map 19246fa) against Big Grassland, with main ab80162 as the baseline and a probe build that plants nothing, to put a number on the trees. Run while Gota was away; every run kept the game window focused (all samples), window 2494 by 1360 (the maximized client area for `FL_WINDOW=2560x1360`), GPU clock median 1860 to 1890 MHz on every run.

Setup: `FL_TEST_FRONT=1 FL_UNITS=100000`, AI on, camera locked, yaw -1.5708. Flank view at x = -982, pitch 0.35, distance 15 (the line's west end, as on Big Grassland); mid-air view at x = -892, pitch 0.28, distance 140. Flank runs 120 s, mid-air 75 s, the mean of the second half of the fps lines. Binaries in `tmp/runs/hill-bench/bin`: `flanks-main` (ab80162), `flanks-hill` (19246fa), `flanks-noveg` (19246fa plus a probe line that returns from `respawn_vegetation` when `FL_PROBE_NO_VEG` is set; the line is not on the branch). Script `work/scripts/perf/hill-bench.sh`, logs in `tmp/runs/hill-bench/`.

| View | Build, map | fps | ms | Drawn | Opaque pass ms | Unit pass ms | Shadows ms |
|---|---|---:|---:|---:|---:|---:|---:|
| flank | main, Big Grassland | 181.5 | 5.51 | 150,627 | 0.48 | 2.81 | 0.39 |
| flank | main, Big Grassland (repeat) | 179.4 | 5.58 | 150,630 | 0.48 | 2.81 | 0.39 |
| flank | hill branch, Big Grassland | 181.7 | 5.51 | 150,619 | 0.47 | 2.80 | 0.38 |
| flank | hill branch, Big Hill | 175.7 | 5.70 | 37,844 | 0.78 | 2.25 | 0.50 |
| flank | hill branch, Big Hill (repeat) | 175.3 | 5.71 | 37,938 | 0.78 | 2.28 | 0.51 |
| flank | no trees, Big Hill | 179.0 | 5.59 | 124,864 | 0.54 | 3.23 | 0.40 |
| mid-air | main, Big Grassland | 175.3 | 5.71 | 182,774 | 1.29 | 2.72 | 0.32 |
| mid-air | hill branch, Big Hill | 158.9 | 6.29 | 182,873 | 1.78 | 2.75 | 0.43 |
| mid-air | no trees, Big Hill | 176.2 | 5.69 | 182,896 | 1.23 | 2.72 | 0.32 |

What it says:

- The branch leaves Big Grassland as main has it: 5.51 ms against 5.51 and 5.58.
- The hill's ground costs nothing: Big Hill without trees runs as Big Grassland (5.69 ms against 5.71 mid-air).
- The trees cost 0.6 ms in the mid-air view (176 to 159 fps), where the same soldiers are drawn: +0.55 ms in the opaque pass and +0.11 ms in the shadows. 1600 trees and 500 shrubs on Big Hill.
- The flank view on Big Hill is not a like-for-like reading. The 200k line fills the width out to x = +-974 and the edge planting starts at |x| = 920, so the camera at the line's end stands among the west outlier trees, which hide three quarters of the army from the occlusion cull (38k drawn against 151k). The same overlap puts the end regiments of the 200k line among the trees: 169 Big Hill plants have crowns over the army's area (559 on Hill). Both go away with the ridge deployment of handoff 120, which has to keep the line inside the planted corridor.

Gota's position before the run: the trees stay; if they are a problem, a perf branch.

## Perf candidates for the trees

The runs above say where the 0.6 ms goes (+0.55 ms opaque pass, +0.11 ms shadows), not which trees or why. Candidates, each with the check that decides it, for a perf branch if one is wanted:

| Candidate | Most it can save | Check that decides it |
|---|---|---|
| Near trees drawn at full detail: the mid-air camera stands over the west edge woods, and their full meshes (overlapping leaf cards) cover much of the screen | most of the +0.55 ms opaque | One run with level 1 forced everywhere, and the count of trees per detail level in that view (`RUST_LOG=flanks::vegetation=debug` logs every level switch) |
| Shadows from far trees and cards | 0.11 ms | One run with levels 1 and 2 not casting shadows |
| Trees the camera cannot see: from low views the ridge hides the rear woods. The soldiers' occlusion cull (PR #31) could test trees too | low views only; mid-air sees the woods | Count how many trees the ridge hides in a low view |
| Per-tree CPU overhead: every plant is a parent entity plus one per mesh part, and `select_tree_level` walks all 2,100 plants every frame. On Grassland 296 plants cost 0.12 ms of main thread (devlog 0163). Instanced drawing as the soldiers have would remove it | frame time once the GPU cost is gone | The CPU frame breakdown with and without trees |

The first two checks come first: about five minutes of runs, and they decide whether a perf branch is needed.
