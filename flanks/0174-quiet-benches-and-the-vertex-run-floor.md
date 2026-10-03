# 0174: Quiet benchmarks, and what the unit pass's floor is made of

Written on Windows 10 in the NTFS tree at `D:\ggando\gamedev\flanks`, for copying into the Linux devlogs. 2026-10-02, 13:15 onward, worked unattended while Gota was away ("use this idle compute to explore the performance situation, branch off main, install whatever profiler"). RTX 3090, Ryzen 9 3900X, display 2560x1440 at 125% plus a 1080x1920 portrait monitor. Windows clone `C:\Users\gotag\projects\flanks`, branch `perf/side-explore` off main 48b3e0c (PR #23 merged). Follows devlog 0173.

## Summary

- The benchmarks were sharing the GPU with the Claude desktop app. While its window is visible, its GPU process and DWM each take 7 to 10% of the 3D engine, game window on top or not. Minimized, under 1% together. Same binary, mid-air view, 200k: 150.8 to 152.4 fps with the app showing, 165.3 to 167.1 minimized, and four minimized runs agree within about 1%. This was most of the "5 to 10% run to run" noise of the memory `windows-bench-noise`. Every run below minimizes it (`tmp\tools\quiet_desktop.ps1`).
- Quiet, main as merged, maximized 2K window: all four views are above 144 fps at 200k (146 to 165). At 300k they read 124 to 133.
- The unit pass is bound by vertex shader runs, about 44 ps each, whatever the shader does: an empty vertex shader with no loads costs as much as the full one. Triangles cost almost nothing on top. Three ways to cut the runs were tried, each exact; none pays on this GPU (below).
- Three small fixes committed on the branch: the Tracy build panicked at startup, the hidden F3 overlay re-shaped its text every frame (about 0.45 ms of the main thread, no fps change while GPU-bound), and the ring pipeline's startup system had no ordering against Bevy's mesh pipeline (a latent panic that adding any RenderStartup system could trigger).
- One exact GPU saving: the soldiers drawn at L3, half of those drawn in the wide views, are posed for their body alone. Pose pass 0.32 to 0.20 ms at 300k, +0.7 to +1.5% fps.
- A GPU frame timer (`FL_GPU_FRAME_TIMER=1`) splits the 0.55 to 0.75 ms the timed passes leave out: 0.17 to 0.28 ms of untimed GPU work inside the frame, 0.35 to 0.54 ms between frames.
- For Gota's look call, quiet at 300k: `FL_LOD_PX=28,12,4` +9 to +10%, `28,12,5` +15%, `36,16,4` +15 to +20%, no shadow on L2 (`FL_RECEIVE_LODS=2`, new) +0.5 to +4%. Side-by-side crops in `tmp\runs\lookb\` in the clone.

## Tools installed

Gota allowed installing a profiler. Nothing went into Program Files, nothing needed admin rights:

- Nsight Systems 2026.5.1, the MSI unpacked with `msiexec /a` into `C:\Users\gotag\profilers\nsys` (1.9 GB). Its Vulkan trace needs its layer registered under HKLM, which needs admin, so only the WDDM trace works (`--trace=wddm`, the account is in Performance Log Users). Wrapper `tmp\tools\nsys_run.ps1`, analysis `tmp\tools\wddm_gpu.py` (3D engine busy time by process, idle gaps). With hardware scheduling on, the game's packets go to a `Graphics_1` node whose DMA packet times do not read as execution times, so the trace answered "who else uses the GPU" (the Claude app, DWM) but not the frame's anatomy.
- Tracy 0.13.1 (matches the tracy-client the game builds against), unpacked into `C:\Users\gotag\profilers\tracy-0.13.1`. `cargo build --profile opt-dev --features tracy`, then `tmp\tools\tracy_run.ps1` (captures from launch, exports aggregate zone stats), `tmp\tools\tracy_zones.py` (filtered per-occurrence zones in a time window, streamed: the full unwrapped export of a 60 s capture runs to several GB), `tmp\tools\tracy_frames.py` (per-frame main and render thread, tick against no tick), `tmp\tools\tracy_rank.py`.
- The downloads stay in `C:\Users\gotag\profilers\downloads` (the 664 MB MSI, the Tracy zip); deleting them is Gota's call.
- A Windows Defender Firewall prompt for `flanks-tracy-fix1.exe` (Tracy opens a port) is open on the desktop. Not answered: allowing or cancelling both change firewall rules. Screenshots now come from the window itself (`tmp\tools\shot2.ps1`, PrintWindow with PW_RENDERFULLCONTENT), so a dialog over the game no longer shows in them.

## The Claude app on the GPU

`tmp\tools\quiet_desktop.ps1 -Action gpu` samples Windows' per-process GPU engine counters. With no game running: Claude 6.9 to 9.7%, DWM 6.9 to 9.6% of the 3D engine while the app streams output. Minimized (`-Action min`): DWM 0.35%, Claude gone. The WDDM trace of a loud mid-air run showed the Claude GPU process with 1323 packets in 4.4 s on the 3D node. The fps follow:

| Mid-air, 200k, same binary | fps | unit pass |
|---|---|---|
| App visible, round 1 | 166.4 | 3.61 ms |
| App visible, round 2 | 150.8 | 3.83 ms |
| App minimized, four runs | 165.3 to 167.1 | 3.60 to 3.68 ms |

The unit pass itself reads 6% longer while the app draws: the GPU is time-sliced between the two. Gota's own numbers come from playing with the app in some state, so they are not comparable to these.

## The quiet baseline (main, 48b3e0c)

| View | 200k fps | 300k fps |
|---|---|---|
| Low flank | 160.1 | 132.6 |
| Mid-air | 164.7 | 132.7 |
| Low, mirrored | 146.3 | 123.9 |
| Mid-air, mirrored | 154.5 | 125.1 |

At 300k the timed GPU passes add up to 6.96 ms (mid-air) and 6.99 ms (flank) against a 7.54 ms frame: about 0.55 to 0.6 ms more per frame than the passes, as at 200k (0.6 ms). 300k at 144 fps needs both less GPU work and that gap closed.

DX12 instead of Vulkan (`WGPU_BACKEND=dx12`, loud): 102.7 fps against 151.6, the terrain pass 3.81 ms against 1.11, the build pass 0.33 against 0.14. Vulkan stays.

## The CPU side (Tracy, mid-air, 200k)

Per frame, means over 30 to 50 s of the battle (Tracy's own overhead included):

- Main thread: Update and PostUpdate 4.7 ms, extract 1.5 ms. A frame that carries a sim tick is main-thread bound: 7.1 ms of main app (the fixed loop 1.7) plus extract, frame 8.3 ms against 5.95 without a tick.
- PostUpdate's critical path was `measure_text_system` (0.17 ms) then `ui_layout_system` (0.32) then `text_system` (0.81), then the visibility systems. `update_overlay` wrote the F3 readouts' text every frame while hidden, so Bevy re-measured, laid out and shaped five lines each frame. Fixed (e75af66): text_system 0.56 to 0.21 ms a frame, measure_text 0.17 to 0.05, PostUpdate 2.71 to 2.25. Quiet A/B, two rounds, mid-air and flank: 166.7/166.9, 165.0/164.1, 159.4/158.2, 158.2/158.0 fps. No change: the frame is GPU-bound; the main thread had room.
- Render thread: 5.7 ms a frame, of which about 2.9 ms is Bevy's prepare and queue systems before the render graph (a chain of small systems, much of it executor overhead under Tracy), 2.45 ms the render graph (pass encoding 1.7, `queue.submit` 0.5), present 0.26.
- `ui_layout_system` still takes 0.39 ms a frame: the hidden card bar's nodes (`Visibility::Hidden` keeps them in the layout) are updated by `refresh_cards`. Not pursued.

How far the CPU is from setting the rate, by taking GPU work away (quiet, mid-air): `FL_LOD_PX=28,12,5` unit pass 3.6 to 2.81 ms, frame 6.0 to 5.37 ms (186 fps); `28,12,10` unit pass 2.14 ms, frame 5.04 ms (199 fps). The frame follows the GPU with diminishing returns: the CPU ceiling at 200k is near 200 fps.

## The unit pass's floor

Probes (temporary shader switches, never committed; `tmp\probes\apply_probes2.py` on top of the night's `apply_probes.py`), quiet, same binary:

| Mid-air, 200k | vertex runs | triangles | unit pass |
|---|---|---|---|
| Full shader, welded | 53.0M | 37.4M | 3.60 ms |
| No raster (every corner moved to one point after the full shader) | 53.0M | 37.4M | 2.41 ms |
| Empty vertex shader: no load, no math, every corner on one point | 53.0M | 37.4M | 2.34 ms |
| Index entry loaded only | 53.0M | 37.4M | 2.32 ms |
| Index, level vertex and pose slot loaded, no math | 53.0M | 37.4M | 2.34 ms |
| Full shader, unwelded (`FL_UNIT_WELD=0`) | 111.8M | 37.3M | 6.25 ms |
| Empty shader, unwelded | 111.8M | 37.3M | 4.95 ms |

The low flank view gives the same picture (3.28 ms empty, 3.32 no raster). So:

- The floor is not the vertex shader's work. Devlog 0173's no-raster probe kept every load and every line of math; with none of them the time is the same.
- Welded against unwelded with the empty shader: 58.8M more vertex runs cost 2.61 ms, 44 ps per run. The same 2.62 ms with the full shader. 53.0M runs at 44 ps is 2.35 ms: the whole floor. The triangles add next to nothing.
- What sits on top of the floor (3.60 against 2.34 ms) is the raster and the fragments.

So the unit pass shrinks with fewer vertex shader runs, nothing else. Memory `unit-pass-triangle-bound` said "fewer triangles"; it is fewer vertices shaded.

## Three exact ways to cut the vertex runs, none of which pays

### 1. Cull the triangles in a compute pass before the draw

An offline estimate (`tmp\tools\cull_estimate.py`, rest pose, the camera's pitch, each level at its band's pixel heights, standard 4x MSAA sample positions): of each level's triangles, only L0 34 to 41%, L1 23 to 33%, L2 13 to 22%, L3 4 to 8% face the camera and cover a sample. The rest the hardware throws away after shading their corners.

Built (local branch `perf/unit-cull-experiment`, df179c6, not pushed): after the pose pass one workgroup per drawn soldier of L1 to L3 poses the level's corners with the same `place()` into workgroup memory, keeps the triangles that face the camera, touch the viewport and cover a sample (margins for the rasterizer's snapping), writes the survivors in welded order into a region of a shared index buffer, and the camera draws the culled list, falling back to the full draw when a level outgrows its room. Exact: the fragments shaded match with it off (6.44 to 6.50M against 6.40M, the difference is draw order) and frozen deployment frames match within the noise floor.

It does what the estimate said to the draw, mid-air: vertex runs 53.0M to 37.0M, unit pass 3.62 to 2.92 ms. But the pass costs 2.4 to 3.8 ms. Per level: L1 1.2 ms, L2 2.0, L3 0.6. With `place()` compiled out the bare structure (workgroup per soldier, vertex loads, barriers, the scan) still costs L1 0.11, L2 0.36, L3 0.44 ms, and the per-triangle tests add 0.6 to 0.7 ms per level; `place()` halves the occupancy on top. Posing every corner again plus testing every triangle costs about what the vertex shader runs it removes cost. On this GPU, culling in compute does not pay. Mesh shaders would do it in the pipeline, but wgpu's are experimental.

### 2. Colour and atlas point per triangle, by primitive index

The far levels' corners are split today so each triangle has a provoking vertex of its own for its flat atlas point (devlog 0172). Moving colour and atlas point to a per-triangle buffer the fragment reads by `@builtin(primitive_index)` welds them on position, normal and part alone: L1 732 to 836 vertices to 452 to 584, L2 260 to 330 to 176 to 252 (`tmp\tools\weld_estimate.py`). In game, mid-air: vertex runs 53.0M to 40.5M as predicted, fragments the same, but the unit pass went from 3.6 to 9.3 ms. With the builtin read but its value unused it is still 9.33 ms: on the NVIDIA Vulkan driver a fragment shader that reads the primitive index puts the draw on a much slower path. Patch kept in `tmp\patches\tri-attrs-experiment.patch`.

### 3. A better provoking vertex assignment

`weld` gives each triangle its provoking vertex greedily in Tipsify order and duplicates a corner when all three are taken. A maximum bipartite matching between triangles and their corners (`tmp\tools\provoke_matching.py`) gives the same counts within a few vertices on every level: the greedy pick is already optimal.

### Also measured, no gain

- The unit pass resolving the MSAA target at its end: Bevy's opaque pass resolves it again over every pixel, so the first resolve is never seen. Dropped (exact, frozen frames within noise), quiet A/B two rounds: 167.2/167.3, 159.2/159.2, 166.4/165.4, 158.8/158.9. An 8-bit target's resolve is nearly free on this card. Not committed.
- The 11.2 MB snapshot upload per tick: uploading one tick in 64 (probe) changed nothing quiet (167.0/167.1, 165.3/166.7).

## What the frame spends outside the timed passes

`FL_GPU_FRAME_TIMER=1` (committed, 06406da) brackets each frame's command buffers with two timestamps, written over two consecutive frames: `render/frame_span` from the frame's first GPU command to its last, `render/frame_gap` from one frame's last to the next one's first. Quiet, 2560x1360:

| | frame | span | gap | passes summed | in the span, untimed |
|---|---|---|---|---|---|
| 200k mid-air | 6.05 ms | 5.67 | 0.46 | 5.39 | 0.28 |
| 300k mid-air | 7.45 | 7.08 | 0.35 | 6.84 | 0.24 |
| 200k low flank | 6.24 | 5.86 | 0.54 | 5.69 | 0.17 |
| 300k low flank | 7.55 | 7.18 | 0.37 | 7.00 | 0.18 |

Span plus gap is the frame within 0.1 ms. So of the 0.55 to 0.75 ms the passes leave out, 0.17 to 0.28 ms is GPU work between or around the passes (the unit pass and Bevy's opaque pass are two render passes on the same 4x MSAA targets, so the colour and depth are stored and loaded again in between; barriers; Bevy's own small passes), and 0.35 to 0.54 ms is between frames: wgpu's staging copies of the frame's buffer writes, which run ahead of its first command buffer, and the GPU waiting.

The swapchain's frame latency (Bevy's `desired_maximum_frame_latency`, default 2), mid-air 200k, two rounds each (temporary switch, not committed):

| latency | fps | gap |
|---|---|---|
| 1 | 110.0, 110.3 | 3.71, 3.52 ms |
| 2 (default) | 165.9, 165.1 | 0.44, 0.37 |
| 3 | 166.6, 167.0 | 0.30, 0.26 |

At 1 the CPU and the GPU take turns, which shows the gap is the GPU waiting. A third queued frame takes 0.1 ms of it for +0.6% fps and a frame more of input lag: not worth it. What stays (0.26 ms) is not the CPU being late.

What a second render pass on the same targets costs, to price merging the unit pass into Bevy's opaque pass: four empty passes that load and store the 4x MSAA colour and depth between the two (probe, not committed), mid-air 200k, two rounds: 167.8 / 167.7 fps to 167.2 / 166.2, frame span 5.45 / 5.59 to 5.54 / 5.60 ms. A few hundredths of a millisecond per pass at most (the targets are compressed): merging would buy nothing worth the surgery.

An attempt to compare windowed with borderless and exclusive fullscreen at the same 2560x1440 failed: the game re-applies the saved window mode (Gota's settings say Windowed) after startup, so the temporary switch ended up in a small window. Comparing would need the settings file changed, which is Gota's.

## The L3 soldiers posed for their body alone (8170100)

The pose pass worked out every slot for every drawn soldier: both legs by IK, the arms, the shield, a spearman's jointed arm, an archer's 14-part bow rig. Every kind's L3 is a single part, the body (all its corners on part 0), and `place()` for part 0 reads only the slots `pose_begin` writes (position, yaw, turn, hip, death, team); an L3 soldier never casts a shadow either (`CAST_LODS` is 2). Half the soldiers in the wide views are L3 (91k of 193k mid-air at 200k, 138k of 289k at 300k).

The build pass now sets the top bit of a soldier's pose source when the camera draws him at a level from which every level of his kind is body only (`PullMeshGpu::body_only`, computed from the meshes, so a future L3 with limbs is posed in full) and he casts nothing; the pose pass returns after `pose_begin` for him. `FL_POSE_BODY=0` turns it off.

300k, 2560x1360, off against on, one binary, two rounds: pose pass 0.31 to 0.33 ms down to 0.19 to 0.21; mid-air 134.1 / 132.9 to 135.0 / 134.9 fps, low flank 132.0 / 130.9 to 133.1 / 132.8 (+0.7 to +1.5%). Frozen deployment frames, mid-air, one binary: on against off 3756 pixels differ, on against on 4511.

## Vertex shader runs per level

Measured by skipping levels (`FL_PROBE_SKIP`), mid-air, 200k: L1 about 1061 runs per soldier for 774 distinct vertices (1.37), L2 364 for 298 (1.22), L3 77 for 66 (1.17; the fallen bodies are in these counts too). A model of NVIDIA's reuse in batches (`tmp\tools\batch_model.py`, `batch_fit.py`, after meshoptimizer's analyzer and Kerbl et al. 2018) fits these best with batches of about 12 triangles, but not closely. Tipsify's cache size, which orders the welded list, barely moves the total: 52.4 to 53.3M runs for cache sizes 6 to 32 against 53.0M at today's 16 (probe, not committed). With every triangle owning its provoking vertex a level cannot go below one run per triangle per batch; a batch-aware order might take L1 and L2 down by 10 to 15% under the model, perhaps 0.1 to 0.2 ms, with the model this uncertain. Not pursued.

## The look candidates at 300k, measured for Gota

Quiet, 300k (`FL_UNITS=150000`), 2560x1360, one run each, battle blocks 15 to 22. `FL_RECEIVE_LODS=n` (new switch, below) keeps the sun's shadow on levels below n only; L0 to L2 receive today.

| Candidate | Mid-air | Low flank | Low, mirrored |
|---|---|---|---|
| Main (`FL_LOD_PX=28,12,3`) | 134.5 | 131.4 | 123.0 |
| `FL_LOD_PX=28,12,4` | 146.4 (+9%) | 144.3 (+10%) | 135.0 (+10%) |
| `FL_LOD_PX=28,12,5` | 154.1 (+15%) | 151.6 (+15%) | 142.9 (+16%) |
| `FL_LOD_PX=36,16,4` | 154.4 (+15%) | 157.6 (+20%) | 144.1 (+17%) |
| `FL_RECEIVE_LODS=2` (no shadow on L2) | 139.9 (+4%) | 132.0 (+0.5%) | 125.3 (+2%) |
| `28,12,4` and no shadow on L2 | 149.9 (+11%) | 144.6 (+10%) | 135.9 (+10%) |

Unit pass, mid-air: 5.08 ms (main), 4.33, 4.01, 3.99, 4.86, 4.15. The L2 threshold is the lever: at 3 px L2 holds 134k soldiers in the mid-air view, at 4 px 89k, at 5 px 62k. The shadow on L2 is worth about 0.2 ms in the mid-air view and nothing in the low ones, where L2 starts past most of the cascades. Side-by-side crops of each in a battle frame: `tmp\runs\lookb\sheet-*.png` in the clone.

## Where that leaves the frame

- 200k: above 144 fps in every view when the GPU is not shared. GPU-bound, CPU ceiling near 200 fps.
- 300k: 124 to 133 fps. The timed GPU work alone is about 7.0 ms, so 144 needs both GPU work off and the 0.55 to 0.6 ms outside the timed passes found.
- The unit pass shrinks only with fewer vertex shader runs. Three exact ways to get them were built and measured above and failed; more were only estimated (below, "What was left untried"). What is certainly left is the look: the level thresholds (devlog 0173 table: +7 to +19%), or lighter far meshes (L2 is 250 triangles on 260 to 330 vertices for a soldier 3 to 12 pixels tall). Both Gota's call.

## Commits on perf/side-explore

| Commit | What |
|---|---|
| f14391c | Add the render diagnostics only where Bevy has not (the Tracy build's startup panic) |
| e75af66 | Write the overlay's text only while it shows |
| 29233fd | Order the ring pipeline's setup after Bevy's mesh pipeline |
| 06406da | Add a GPU frame timer to the periodic log (`FL_GPU_FRAME_TIMER=1`) |
| 7706020 | Add FL_RECEIVE_LODS to limit the levels that receive the sun's shadow |
| 8170100 | Pose the soldiers drawn at L3 for their body alone |

Commit messages trimmed to the writing guidelines (one to three sentences) before the push, which rewrote the hashes; the trees are the same. Pushed to origin as `perf/side-explore` at 8170100 on Gota's word, origin/main still 48b3e0c, so the merged tree is the branch tip: strict clippy clean and the 29 tests pass on it, and the tip's runs logged no shader or validation error. PR draft `work/drafts/038-pr-side-explore.md`; Gota opens it.

Local branch `perf/unit-cull-experiment` (df179c6): the compute cull, a record only, not pushed.

## How the afternoon went, and what was left untried

Order of work, 13:15 to about 17:25: read the handoff and devlog 0173; downloaded and unpacked Nsight Systems (its Vulkan trace refused without admin, the WDDM trace worked and showed the Claude app on the GPU); DX12 against Vulkan; the upload probe; the Claude app finding and the quiet baseline; Tracy (its build panicked: fixed), the overlay text fix; the empty vertex shader probes; the compute cull (three hours with its debugging: the ring pipeline's startup order surfaced as a panic, the cull first gated on a main-world resource and never ran, early battle blocks misread as lost fragments); the per-triangle colour weld; the provoke matching; the MSAA resolve; the frame timer, latency and window mode probes; the look candidates; the L3 body poses; the per-level counts and the Tipsify probe; the extra render pass probe; the write-up.

Mishaps: a full unwrapped Tracy export (`tracy-csvexport -u`) reached 5 GB on a C: with 13 GB free before it was stopped and the file overwritten with the aggregate stats; the export now always filters. The first screenshots copied the screen and caught the firewall dialog and the taskbar; `shot2.ps1` captures the window itself.

The stopping point was soft: the prompt set no target, and the work stopped at about 17:15 with the exact ideas that were cheap to try used up, while Gota was still away. Untried, with what each might give:

- A triangle order built for NVIDIA's vertex batches (meshoptimizer's batch-aware optimizer, or a greedy that fills each batch of about 12 triangles from one patch). Under the batch model L1 and L2 could lose 10 to 15% of their runs, 0.1 to 0.2 ms at 200k. The model fits the measured counts loosely, but trying an order in game costs about an hour and the pipeline statistics give the answer directly. Exact.
- A lighter L2 made by simplifying the current one offline (quadric decimation that keeps part boundaries and the atlas points), shown to Gota against the thresholds above. L2 is 250 triangles for a soldier 3 to 12 pixels tall; at 120 triangles the mid-air view would lose about 12M runs, about 0.5 ms. A look change, model work otherwise done by Astra, but a prototype to judge costs an afternoon.
- One shadow tap instead of the filtered lookup for L2 rather than no shadow: most of the 0.2 ms mid-air at 300k with less visible change than `FL_RECEIVE_LODS=2`. A look change.
- The terrain candidates of devlog 0173 (one patch instead of three past some distance, lower anisotropy far away, the meadow band's mask built at load): 0.3 to 0.5 ms from mid-air, guessed, each a look change. None was built for Gota's eye.
- Mesh shaders, which would cull per triangle inside the pipeline: wgpu's are experimental; not looked into.

With a concrete target (300k at 144 fps in the four views, exact only or with look prototypes allowed) the first three would have been built rather than listed.

## Takeaways

- Measure with the Claude app minimized. Its drawing cost 10% of the frame rate and was most of the run-to-run noise every earlier A/B fought against.
- The unit pass pays per vertex shader run, about 44 ps on the 3090, whatever the shader does. Estimate a change by the runs it removes, from the pipeline statistics in the log.
- Work moved off the hardware path costs about as much in compute as it saves there: per-triangle culling in compute lost by a factor of three.
- A builtin can cost more than the work it replaces: `primitive_index` in a fragment shader is a slow path on NVIDIA's Vulkan driver (2.6x the unit pass).
- Look for work done for nobody: the hidden overlay re-shaped every frame, L3 soldiers were posed limb by limb. Both are exact savings found by reading what each level actually reads.
- Name the frame's untimed time before guessing at it: the frame timer split it into 0.17 to 0.28 ms in the frame and 0.35 to 0.54 ms between frames in one run.
- A system that copies a resource another plugin makes at RenderStartup must be ordered after that plugin's set; the incidental order holds until someone adds a system.
- Give an unattended perf session a target and a stop condition, or it stops when the easy exact ideas run out.

## For the next perf branch

What this afternoon taught about how to work, and the items it leaves, ranked.

### How to work

1. Check the GPU is the game's alone before any run: `tmp\tools\quiet_desktop.ps1 -Action gpu` with no game running should show nothing above a fraction of a percent. A visible Claude app costs about 10% of the frame rate and most of the run-to-run noise.
2. Find what bounds the frame before optimizing anything. Three readings, each a few minutes:
   - `FL_GPU_FRAME_TIMER=1`: the GPU's span per frame against the gap between frames. A large gap means the GPU waits on the CPU.
   - Take GPU work away (`FL_LOD_PX=28,12,10`): the frame rate that remains is the CPU ceiling, about 200 fps at 200k today.
   - The pacing lines (frames with and without a tick, main and render thread).
   Work on whatever is not bounding the frame buys no fps: the overlay fix freed 0.45 ms of main thread and moved nothing.
3. Price an idea with a probe that removes the work wholesale before building it. The empty vertex shader settled what the unit pass costs in ten minutes; the empty extra pass priced merging the passes at almost nothing; the skipped upload priced the snapshot at nothing. The compute cull should have been priced the same way before three hours of building it: a dispatch of one empty workgroup per soldier would have shown 0.4 to 0.9 ms of bare structure at once.
4. Count the unit pass's cost in vertex shader runs, from the pipeline statistics in the log, about 44 ps each on the 3090. Per level, skip the others (`FL_PROBE_SKIP`). What the vertex shader computes or outputs is free; what lowers the runs is fewer or better shared vertices.
5. Look for work done for nobody: read what each consumer actually reads and stop producing the rest. The hidden overlay's text and the L3 soldiers' limb poses were both found this way, and both are exact.
6. Check exactness two ways: frozen deployment frames against a noise floor of two runs of the same binary (`tmp\probes\exe_imgcheck.ps1`), and in battle the fragment shader count of the pass, which must not drop. A frozen frame where the change is not active proves nothing (the first cull check passed because the levels had fallen back to the full draw).
7. A/B one binary with an env switch where possible: same build, same layout, alternate rounds, and the switch stays for the next person.
8. Do not redo hardware work in compute on this GPU: posing and testing in a compute pass cost as much as the vertex runs it saved. And check a feature's cost before leaning on it: `primitive_index` in a fragment shader is a slow path on NVIDIA's Vulkan driver, DX12 is far slower than Vulkan here.
9. Keep the C: drive in mind: never export a whole Tracy capture unwrapped, copy binaries only for an A/B in progress.

### Where the 300k frame goes

Mid-air, 300k, quiet: frame 7.45 ms, the GPU busy 7.08 of it. Unit pass 5.04 (70%), terrain 0.91, pose 0.35 (0.20 with the L3 change), sun shadows 0.24, build 0.18, small passes 0.1, untimed in the span 0.24; between frames 0.35. 144 fps needs 6.94 ms: about 0.5 ms off, nearly all of which has to come from the unit pass.

### Items, ranked by what they buy toward 300k at 144

Exact, build and measure:

1. A triangle order built for NVIDIA's vertex batches, for L1 and L2. Today L1 runs 1.37 times its distinct vertices and L2 1.22; Tipsify's cache size does not move it, so the order has to target batches of about 12 triangles rather than a FIFO cache (meshoptimizer's approach, or a greedy that fills each batch from one patch). Guess: 10 to 15% fewer runs on those levels, 0.1 to 0.2 ms at 200k, 0.2 to 0.3 at 300k. About an hour to a first answer: the weld's order is one function, and the log gives the runs.
2. The tick frame's render thread: the snapshot's 1 ms copy into the staging buffer lands on one frame in five and leaves the GPU waiting. Copy in parallel or shrink the record (56 bytes, the colour and seed could be 8). Part of the 0.35 ms gap at most; the skipped-upload probe found nothing, so low.
3. The hidden card bar out of the layout (`Display::None` instead of `Visibility::Hidden` while the HUD is off): 0.39 ms of main thread. No fps while GPU-bound; it matters once the look items move the frame toward the CPU ceiling.

Look, as prototypes with crops for Gota's eye:

4. A lighter L2. L2 is 250 triangles on 260 to 330 vertices for a soldier 3 to 12 pixels tall, and the largest single cost. Simplified offline to about 120 triangles, keeping part boundaries and atlas points: about 12M fewer runs mid-air at 200k, about 0.5 ms, more at 300k. Model work otherwise, but a prototype from the current mesh takes an afternoon.
5. The level thresholds: `FL_LOD_PX=28,12,4` alone gives +9 to +10% at 300k, enough for 144 in the mid-air and low flank views; `36,16,4` +15 to +20%. Crops in `tmp\runs\lookb\`.
6. One filtered tap of the sun's shadow for L2 instead of the full lookup: most of the 0.2 ms that removing it buys mid-air, with less change than no shadow.
7. The terrain candidates of devlog 0173 (one patch beyond some distance, lower anisotropy far away, the meadow band's mask built at load): 0.3 to 0.5 ms from mid-air, guessed.

A bigger bet:

8. Mesh shaders. Per-triangle culling inside the pipeline is what the compute cull tried to do outside it: L2 keeps only 13 to 22% of its triangles from the views players use. wgpu's mesh shaders are experimental and the platforms are uneven, so a spike first: one level drawn by a mesh shader on Vulkan, its runs and time against the vertex path.

Tried and not worth repeating: culling in a compute pass, colour per triangle by primitive index, a matched provoking vertex weld, Tipsify cache sizes, skipping the unit pass's resolve, merging the unit pass into Bevy's opaque pass, a third queued frame, DX12, skipping the snapshot upload.

## Index row for devlogs/README.md

| 0174 | 2026-10-02 | [Quiet benchmarks, and what the unit pass's floor is made of](0174-quiet-benches-and-the-vertex-run-floor.md) | An afternoon on Windows with Gota away. The Claude app's drawing took 10-20% of the GPU during every earlier benchmark (151 against 167 fps); measured with it minimized, 200k is above 144 fps in all four views, 300k reads 123-134. The unit pass costs 44 ps per vertex shader run whatever the shader does; compute culling (exact, three times its saving), colour per triangle by primitive index (NVIDIA slow path) and a matched weld all failed. Branch perf/side-explore, pushed: L3 soldiers posed for their body alone (pose pass 0.32 to 0.20 ms at 300k), a GPU frame timer, FL_RECEIVE_LODS, the Tracy build, the hidden overlay text and the ring pipeline's startup order fixed. Level thresholds measured at 300k for Gota's call: +9 to +20% |
