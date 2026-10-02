# 0171: What limits the frame after the unit pass: GPU vertices at 1440p, and the sim sharing the frame's task queue

Written on Windows 10 in the NTFS tree at `D:\ggando\gamedev\flanks`, for copying into the Linux devlogs. 2026-10-01. RTX 3090, Ryzen 9 3900X. Windows clone at `C:\Users\gotag\projects\flanks`, on `feat/hover-rings-toggle` (main 8d1f7f4 merged in). Nothing built, nothing run, no source touched.

## The question

After PR #19, Gota reads 108 to 110 fps in a 200k battle with the window maximized on the 2560x1440 display, from his usual spot, and 90 and up in heavier views. The goal: 130 to 144 fps (the display's refresh) in the 200k to 300k battles players will run daily, and 500k at 120 fps or more later. The previous handoff named the sim as the next bottleneck. This entry looks at the whole frame again.

## Method

No runs. Three sources:

- The code of main 8d1f7f4, and the Bevy 0.19, bevy_tasks 0.19 and async-executor 1.14 sources in the cargo registry.
- The frame pacing and GPU pass lines of the PR #19 benchmark logs already on disk (`tmp\runs\bench\`, binary 9cfe068, battle blocks 15 to 34, the same window as devlog 0170). `tmp\tools\frame_anatomy.py <log> [first] [last]` averages them.
- The unit GLBs, read with the GLB reader of devlog 0170: `tmp\tools\far_level_uv.py`, `far_level_normals.py`, `far_level_sharing.py`.

## The frame, from the existing logs

GPU busy is the sum of the timed GPU passes over the mean frame. The frame columns come from the periodic pacing lines: plain frames carry no sim tick, tick frames do. Update+PostUpdate is the main thread from the end of the fixed loop to `Last`.

| Run (2560x1360 unless noted) | fps | GPU passes | unit pass | GPU busy | plain frame p50 / p90 | tick frame p50 | fixed tick p50 | Update+PostUpdate p50 / p90 |
|---|---|---|---|---|---|---|---|---|
| Saved 200k battle, AI, flank camera | 108.3 | 8.28 ms | 6.84 ms | 90% | 6.5 / 14.2 ms | 10.4 ms | 1.65 ms | 3.2 / 10.5 ms |
| Flank clash | 92.9 | 9.74 | 8.49 | 90% | 9.8 / 14.3 | 11.1 | 1.32 | 2.9 / 10.8 |
| Flank clash, 1600x900 | 127.6 | 6.35 | 5.30 | 81% | 5.7 / 13.9 | 7.7 | 1.35 | 3.1 / 10.8 |
| Flank clash, `FL_LOD_PX=40,18,8` | 138.7 | 5.50 | 4.39 | 76% | 5.8 / 13.2 | 7.0 | 1.40 | 3.1 / 10.8 |

The saved battle is the closest to Gota's own check (108.3 against his 108 to 110). Per two seconds it has about 157 plain frames and 60 tick frames.

## Reading 1: at Gota's window the GPU is still the first limit

At 2560x1360 the GPU passes fill 90% of the frame in both views, and the unit pass is 80 to 87% of the GPU frame. Removing every CPU stall would lift the saved battle only to the GPU's ceiling, about 117 to 120 fps. The sim is the first limit only where the GPU is lighter: at 1600x900 (81% busy) and with coarser levels (76%).

## Reading 2: frames stall behind the sim's chunk tasks

Update+PostUpdate has a p90 of 10.5 to 10.8 ms in every run, at both window sizes and every GPU load, against a p50 of about 3 ms. The plain frames show the same: p50 5.7 to 6.5 ms, p90 13 to 14 ms. So roughly one plain frame per sim tick takes about 8 ms longer, and the cause does not depend on the render load.

The mechanism, from the sources:

- The tick job's integrate spawns one task per 2048 soldiers (`run_tick_job`, src/sim/mod.rs:453), about 98 tasks at 200k, each about 1 ms of work. The grid rebuild and the density field spawn theirs the same way. All go through `util::sim_scope` (src/util.rs:65) onto Bevy's `ComputeTaskPool`.
- Bevy's multi-threaded schedule executor runs every `Send` system as a task on that same pool (bevy_ecs 0.19 `schedule/executor/multi_threaded.rs:274` and `:685`). The render world's systems run there too.
- async-executor 1.14: a worker that takes a task from the global queue also moves half of what is left into its own local queue, and works its local queue first (`Runner::runnable` and `steal`, lib.rs:995 and :1066). With 98 chunks queued, the first workers each take a large batch. A system task queued after them runs only when some worker has emptied its batch, or on the fairness steal every 64 tasks.
- A system that opens a plain `ComputeTaskPool::scope` and waits in it runs queued pool tasks meanwhile, sim chunks included. `pack_soldier_snapshot` (src/render_units_gpu.rs:270) does this on every tick frame, right after the job is kicked.

The job is kicked at the end of the fixed tick and runs about 10 ms of wall time on the pool's 16 workers (grid 2 to 3, field 0.6 to 0.8, kernel 6 to 7 ms in these logs). The tick frame's Update and the frame after it run inside that window.

History. Devlog 0079 measured the same mechanism and built `FL_SIM_POOL`, a second pool of 12 threads for the sim. It was dropped: on top of Bevy's 16 workers it put 28 busy threads on 24 hardware threads and worked through OS scheduling fairness, which would invert on a smaller machine. Devlog 0126's band-parallel grid rebuild made the frame slower for the third reason it lists, the rebuild's tasks occupying the shared pool. Both are this mechanism.

## Reading 3: tick frames carry 2 to 4 ms more

Tick frames are 2.0 ms (900p) to 3.9 ms (saved battle) longer than plain frames. On the main thread: the fixed tick itself (1.3 to 1.65 ms p50), then `pack_soldier_snapshot` (11.2 MB written, in a scope that can run sim chunks). On the render thread: `prepare_gpu_units` copies the same 11.2 MB snapshot into `write_buffer`. The job's first tasks start during this frame.

O(N) loops in the fixed tick on the main thread (from a catalog of every registered system):

- `update_groups` (src/frontline.rs:352): one pool task per regiment, every soldier, with random reads through `target`.
- `fill_vacated_slots` (src/combat.rs:161): a serial scan of all soldiers on any tick a formed regiment lost a man, then a scan and sort per hole.
- The alive count in `process_deaths` (src/combat.rs:145): a serial scan of two byte columns every tick.
- `apply_reforms` to `assign_slots` (src/formation.rs:611, :122): a serial scan of all soldiers per reforming regiment.
- `neighbour_audit` every 60 ticks (src/sim/diag.rs:40): 200k grid queries for the overlay's nn and move readouts, the "audit" figure in the stats line, about 1 ms every 2 s. It runs in player builds.

## The far levels can share vertices

L1 to L3 draw as expanded corner lists, three vertex shader runs per triangle, because their GLB vertices barely repeat (1.1 corners per vertex). Devlog 0170 concluded an indexed draw gains nothing there. The GLBs say why the vertices do not repeat:

- The atlas coordinate is the same on all three corners of every far-level triangle (100% of L1 to L3 triangles in all four kinds, extent 0.0 texels). Each face samples one atlas point, and neighbouring faces sample different points, so the exporter splits every vertex per face.
- The normals are per corner, not per face: only 0 to 46% of far-level triangles have one normal on all three corners. The far levels are lit smoothly. (The comment in `unit_output` that normals are per face describes the code-built meshes, not the imported ones.)
- Colour and part are constant per triangle.

So the split comes from one per-face value, the atlas point. Welding the corners on (position, part and pivot, normal, colour) and giving each triangle a provoking vertex of its own that carries its atlas point (`@interpolate(flat)` on the atlas coordinate, WGSL's default first vertex) keeps every value the shader sees today: positions, normals and colours per corner as now, and the atlas point as now, since it never varied across a triangle. A vertex is duplicated only where all three corners of a triangle already provoke others.

Vertex shader runs per triangle under a 16-entry FIFO model after Tipsify (`far_level_sharing.py`):

| Level | Triangles | Today | Indexed, shared vertices | Fewer runs |
|---|---|---|---|---|
| L1 | 664 to 680 | 3.0 | 1.36 to 1.55 | 1.9 to 2.2x |
| L2 | 248 to 250 | 3.0 | 1.29 to 1.50 | 2.0 to 2.3x |
| L3 | 48 to 56 | 3.0 | 1.29 to 1.98 | 1.5 to 2.3x |

For comparison, L0 indexed with Tipsify runs at 1.23 to 1.37 (devlog 0170).

Instances. Drawing one instance per soldier, as the indexed L0 does, pays about 8 ns per instance (devlog 0078): 1.3 ms for the 166k far soldiers of the flank clash. The pulled draw avoids that today. The indexed far draw keeps it avoided with an index buffer holding K copies of the level's list, copy k offset by k times the level's vertex count, and one instance per K soldiers: the soldier is `instance * K + vertex_index / V`, the corner `vertex_index % V`. The hardware's reuse works on index values, so it holds inside each soldier. The finalize pass writes the instance count, and the unused tail of the last group draws degenerate triangles.

Expected, an estimate: at 2560x1360 in the flank clash the far levels run about 105M vertex shader invocations a frame (L1 26M, L2 66M, L3 13M) and would run about 50M. Devlog 0170's L0 indexing removed about 22M runs for 1.2 ms, 55 ps a run (its two steps alone read 29 and 78 ps). That puts the saving at 1.5 to 3 ms of the 8.5 ms unit pass, and 1.3 to 2.4 ms of the saved battle's 6.8 ms. The shadow casters (L2 in cascade 0, L3 in cascades 1 and 2) can take the same lists. A variant that fetches the atlas point per fragment through `primitive_index` would share more (L2 1.04 to 1.28 runs per triangle) but needs a wgpu feature, so it stays a variant.

## Smaller items found in the code

Each is small; together they may matter at 144 fps. Their share of the 3 ms Update+PostUpdate median is not known without a trace.

- About 58 `std::env::var` lookups per frame in battle. `scripts_active()` (src/game_state.rs:206, 17 lookups) runs every frame from both `ai_think` and `auto_engage` before their 1 and 2 s timers; the test logs and `apply_camera_transform` add the rest. On Windows each is a lock, an OS call and a UTF-16 conversion.
- `extract_instance_data` (src/render_units.rs:382) copies every instance bucket every frame with no change check, and `prepare_instance_buffers` (:1223) uploads them again. On the GPU path that is the live arrows and the stuck-arrow ring, up to 50k x 64 B = 3.2 MB twice a frame.
- `update_overlay` rewrites its text every frame while hidden, `update_banners` writes about 600 Transforms every frame, `update_balance` resizes a UI node most ticks: each makes Bevy redo layout, propagation or extraction.

## Proposed order

1. Keep the sim from flooding the frame's queue, inside Bevy's one pool. The integrate, the grid passes and the field passes run as at most K tasks, each pulling chunk indices from a shared counter and yielding between chunks; K is the pool's workers less a reserve for the frame (12 of 16 here), with an `FL_` switch to change it. Each chunk writes its own output slot as today, so `FL_HASH` stays equal. No added threads: this is the in-pool form devlog 0079 named after dropping `FL_SIM_POOL`. The job takes longer, about 12 ms of its 33 ms window. Measure: Update+PostUpdate p90 and plain-frame p90 should fall to about their p50; fps in the saved battle and the flank clash at 2560x1360 and 1600x900. Expected: the GPU's ceiling, about 117 to 120 fps in the saved battle at 2560x1360, about 150 in the 900p flank clash. Fallback if the frame still waits: two pools sized to the hardware together (Bevy's compute pool reduced through `TaskPoolOptions`), which does not oversubscribe as `FL_SIM_POOL` did.
2. Index the far levels on shared vertices with K soldiers per instance. Look-neutral; check with forced-level screenshots against the current draw. Expected: 1.5 to 3 ms off the unit pass at 2560x1360.
3. Flatten the tick frames: the snapshot written by the job's chunks (each chunk packs its rows after the kernel), patched on the main thread only for the rows the apply and the death sweep touch; `update_groups`' sums computed by the job from the same kick-time columns in the same order; `fill_vacated_slots` and the alive count from the regiment runs; `neighbour_audit` only with the debug overlay.
4. The smaller per-frame items above, after one trace ranks them.

With 1 and 2 the estimate for the saved battle at 2560x1360 is a GPU frame of about 6 to 7 ms and a CPU frame of about 6.5 to 7 ms averaged over tick frames: about 140 to 150 fps; the flank clash about 125 to 145. Item 3 then takes the 2 to 4 ms off one frame in four or five. All estimates, from the logs above.

## Toward 300k and 500k

- The job grows with the army. At 300k with K = 12 it would take about 18 ms of its 33 ms window. At 500k, about 30 ms: too close. Plan 013 (the band rebuild without the serial merge, the pair pass) is the sim work for that range, and item 1 removes the risk devlog 0126 ran into, the rebuild's tasks slowing the frame.
- The main-thread work per tick grows with the army too (item 3 is needed above 200k whatever happens to the kernel).
- The GPU at 500k: about 2.5 times the far-band triangles. The look decisions stay the levers: the level thresholds, an L3 with legs and weapon at 60 to 80 triangles (spec 009), sprites for the far band.
- Plan 010 item 9 (2026-09-20): the map holds about 360k in formation. Devlogs 0136 to 0167 are not on this machine, so whether that changed is not checked here.

## Not measured

Every estimate above. The 3 ms Update+PostUpdate median is not explained by the code read (no per-soldier loop runs on the GPU path every frame). The render thread's own CPU time is unknown: in every log it is gated by the GPU. One Tracy capture of the saved battle at 2560x1360 (`--features tracy`) would answer both and confirm the queue mechanism directly.

## Index row for devlogs/README.md

| 0171 | 2026-10-01 | [What limits the frame after the unit pass](0171-what-limits-the-frame-after-the-unit-pass.md) | From the PR #19 logs and the code, no runs. At 2560x1360 the GPU is 90% busy, so it still caps Gota's 108 fps. On top, one plain frame per sim tick stalls about 8 ms (Update+PostUpdate p90 10.5 ms at every resolution) because the job's ~98 chunk tasks share Bevy's task queue with every frame system. The far levels repeat almost no vertices only because of a per-face atlas point; welded with a provoking vertex they run 1.3 to 1.55 vertex shader invocations per triangle instead of 3. Proposed: cap the sim at K of the pool's workers, index the far levels with K soldiers per instance, flatten the tick frames |
