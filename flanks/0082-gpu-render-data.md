# 0082: Item 2, the unit render data moves to the GPU (2026-09-22)

Branch `perf/gpu-render-data`, from `main` 347bc5a. Plan: docs/plans/011-gpu-render-data-item2.md, approved by the owner the same day. Background: devlog 0081 (where the CPU frame went), devlog 0078 (vertex pulling), docs/plans/010-scale-to-1m.md items 2 and 3. Reference code absorbed: `exp/vertex-pull` (47f658a), unmerged.

## Commits

- 57a1bf1: turn off wgpu's GPU-side validation of indirect draws (see the trap below).
- 9174248: the GPU path behind `FL_GPU_SYNC=1`, living soldiers only.
- 8886865: the render thread's frame time in the periodic log.
- e5afc05: the fallen on the GPU.
- 5a8963f: the bucket counts read back to the overlay and `FL_GPU_CHECK`.

## What was built

`FL_GPU_SYNC=1` replaces the per-frame CPU sweep with a per-tick snapshot and a per-frame compute pass.

- Once per sim tick, `pack_soldier_snapshot` (PostUpdate, only when `Units` changed) packs a 56 byte record per soldier: position, previous position, facing, previous facing, kind, regiment, swing state and timer, hit flash, death timer, color and seed. Parallel over 16k chunks. The `Vec` is double buffered against the render world and extract SWAPS it, so the handoff is a pointer exchange.
- Every frame, `build_frame_params` copies the top of the old sweep: frustum planes, camera, overstep, the three smoother coefficients, the level bands per kind, and one 24 byte record per regiment (stance tier, cheer progress, march and wall flags, walk phase, broken/selected/hovered flags). The per-regiment expressions moved into shared helpers in render_units.rs so both paths read one definition.
- The compute shader `unit_build.wgsl` is the step-by-step port of the sweep: EMAs before the cull, sphere cull against five planes, jittered level with hysteresis held in a per-soldier GPU buffer, the color tints, walk amount, wrap-aware yaw, the lunge encoding, fx, stagger. It writes the same 64 byte `InstanceData` the vertex shader always consumed and appends `record_index | lod << 30` to the bucket's index list with an atomic counter. A 16 thread `finalize` turns the counts into `DrawIndirect` arguments.
- Index list layout: a soldier of kind k can only land in one of k's four levels, so bucket (k, l) gets `kind_cap[k] + CORPSE_CAP` slots. One dispatch, no prefix sum, 16 bytes per soldier.
- Draw: the sixteen bucket entities keep `NoAutomaticBatching` and get a `PullMesh` (the level mesh with its index list expanded). The pulled pipeline goes through `SpecializedRenderPipelines` with the mesh layout, the corner count and the bucket id in the key. The draw command sets group 3 (records, index list, corners, bucket table) and issues `draw_indirect` at `bucket * 16`. The vertex shader's pose function is untouched; the new `vertex_pull` entry builds the same `Vertex` from storage buffers. The level debug tint moved into that entry behind a shader def.
- Bevy 0.19 specifics: the render graph is schedules now. The compute pass is a plain system in `Core3d`, `.after(Prepass).before(MainPass)`, recording into the frame's encoder through `RenderContext`. Extract uses `ResMut<MainWorld>` to swap the snapshot. The plugin's `finish` reads the adapter and falls back to the CPU path without vertex stage storage buffers.
- The fallen: `Corpses::push` also records the ring slot and the frozen record in a pending list. Extract drains it every frame in both modes, prepare sorts by slot and uploads runs of consecutive slots with one `write_buffer` each into the corpse region at the tail of the records buffer. The compute pass gets one thread per body: cull, level from the plain thresholds, append.
- Counts: `finalize` also stamps the 32 counters, the frame number and the soldier count into a `ShaderBuffer` asset. Bevy's `GpuReadbackPlugin` copies it back and triggers `ReadbackComplete` on a main-world entity one to two frames later. The observer fills `RenderCounts`, so the overlay's drawn and fallen counts work on the GPU path.
- `FL_GPU_CHECK=1`: the CPU sweep runs next to the GPU path, records its per-bucket living and fallen counts by frame number in a short ring, and the observer compares each readback with the entry of the same frame. The first five mismatches are logged in detail, then a summary every 300 frames.

The CPU path is untouched apart from the shared helpers and stays the default. Arrows keep their instanced draws.

## The wgpu indirect validation trap

The first GPU run drew the right picture at exactly 60 fps while the GPU passes read under 0.3 ms and vsync was off. The same binary at 2k soldiers ran at 300 fps, and 20k ran at 300 fps with `WGPU_VALIDATION_INDIRECT_CALL=0`.

Cause: wgpu validates indirect draw arguments on the GPU when `InstanceFlags::VALIDATION_INDIRECT_CALL` is set. Bevy removes that flag in release builds, but only when DX12 is not among the enabled backends, and Bevy's default backend set is `Backends::all()`, which names DX12 on Linux too. So the flag stays on. wgpu then injects a validation compute pass with a fresh staging buffer into every render pass that draws indirectly, and on this box that pinned the frame to the display refresh.

Fix (57a1bf1): main.rs builds `WgpuSettings::default()`, removes the flag, and applies the environment again so `WGPU_VALIDATION_INDIRECT_CALL=1` can turn the check back on. Our arguments come from our own compute shader, from counters bounded by the buffer layout.

Two other first-run lessons: `smooth` is a reserved word in WGSL (the smoothing buffer is `smoothing`), and a Bevy `ComputePipelineDescriptor` in 0.19 wants `immediate_size` and `zero_initialize_workgroup_memory` filled in.

## Measurements

Recipe as in the handoff: `FL_TEST_FRONT=1 FL_UNITS=100000 FL_AI=0 FL_ENEMY_STATIC=1 FL_VOLUME=0 FL_CAM_LOCK=1`, 197k to 200k alive, 900 m view with `FL_NO_CULL=1`, 1600x900, medians from the periodic log. The new `render thread frame` field is the wall time of the render thread's Render schedule.

| 900 m, culling off, 200k | CPU path | GPU path |
|---|---|---|
| Average fps | 53 to 57 | 101 to 155 |
| Frame without a tick, p50 | 15.7 to 15.8 ms | 4.8 to 5.8 ms |
| Frame with a tick, p50 | 17.1 to 17.7 ms | 9.6 to 10.5 ms |
| Main thread, update and post-update, p50 | 11.3 to 12.0 ms | 2.7 to 4.4 ms |
| Extract copy | 1.8 to 1.9 ms | 0 |
| Wait for the render thread, p50 | 2.4 to 2.5 ms | 0.8 ms |
| Render thread frame, p50 | 6.2 to 6.4 ms | 2.8 to 3.4 ms |
| `write_buffer` on the render thread | 1.4 to 1.5 ms every frame | about 1 ms on tick frames only |
| GPU unit pass | 3.3 to 3.5 ms | 1.0 to 1.1 ms plus 0.1 ms of compute |
| Process CPU over 20 s | 100.9 s user, 5.0 cores | 92.3 s user, 4.6 cores |

Reading: the frame halves and the process uses less CPU while doing it. Per frame that is about 87 core-milliseconds on the CPU path against about 35 on the GPU path. The main thread's per-soldier work is gone from every frame. What is left on tick frames is the 6 ms fixed loop (install, serial apply, the 12 MB of prep copies) and the sim job's contention on the compute pool, both outside this item.

The unit pass cost dropped 3.3 times with all 197k soldiers on L3, in line with devlog 0078's pulled measurement. The GPU is now idle for most of the frame. GPU clock was not pinned in these runs; the compute and pull numbers are at whatever clock the light load allowed, so they overstate the cost if anything.

Screenshots at 280 m, same deterministic battle, both paths: identical placement, colors, banners and poses. The direction test, cropped and enlarged 30 s in: the same fallen bodies among the standing ranks on both paths.

### The check mode

| Run | Frames compared | Frames that differed | Worst bucket difference |
|---|---|---|---|
| Direction test, 30 s, 2.2k soldiers, hundreds fallen | 1800 | 0 | 0 |
| 200k at 280 m, 20 s, mixed L2 and L3 | 1200 | 2 | 1 |

The two differing frames each had one extra living soldier in one bucket on the GPU side and the same totals otherwise, so a soldier at the frustum edge went the other way of the `<= 0` test by float rounding. Levels, smoothers and the fallen agree exactly. This is the count-level proof the design asked for. The pose-level proof is the owner's A/B.

Behavior gate: `hashrun.sh dir-gpu 40 FL_TEST_DIR=1` on the GPU path equals the pipelined baseline 19 of 19.

### Owner feel pass (2026-09-22 evening)

In his words, on the 200k scenario: it feels faster than before. Past 60 fps the difference is harder to notice clearly, but the CPU version had moments where it briefly dipped to 30 to 40 fps and clearly lagged, which did not occur on the GPU version. Most of the time it held 100 to 120 fps. It dips to 80 to 90 fairly often, and to about 60 in the worst case on a big camera movement such as suddenly zooming out over the whole battlefield.

Reading: the 100 to 120 is the average of tick frames near 10 ms and the frames between them near 5 ms. The 80 to 90 is the sim job's heavier ticks in melee. The camera dips are a separate question, measured below with a scripted zoom sweep.

### Camera jumps and army size

`FL_CAM_LOCK=1 FL_CAM_SWEEP=4` jumps the locked camera between 40 m and 900 m every 4 s, the worst case of the owner's sudden zoom-out. At 200k the worst frames over 26 s: GPU path 27 to 34 ms, CPU path 35 to 63 ms. The spikes are on both paths and older than this branch, so they are not chased here. The owner reads them as needing continuous camera movement, out of scope for this branch.

Scaling on the GPU path, 900 m, culling off: main thread update and post-update p50 2.3, 2.3 and 2.8 ms at 50k, 100k and 200k soldiers, extract 0 at every size. The render CPU cost stops growing with the army, which was the pass criterion.

### The indexed pull is not built

The design gated it on the 40 m close-up: build it only if it could save over 0.5 ms there. Measured with work/scripts/gpu-pass-measure.sh on the GPU path, 200k battle, 2.9k to 5.4k soldiers drawn in the view:

| 40 m close-up | Unit pass | GPU clock during the samples |
|---|---|---|
| Normal levels (about 1.3k L0, the rest L1) | 0.28 to 0.54 ms | 600 to 1290 MHz |
| Every soldier forced to L0 (`FL_LOD=0`) | 0.26 to 0.58 ms | 690 to 1290 MHz |

The whole pass sits under 0.6 ms at a third of the full clock, so under 0.3 ms at full clock. A non-indexed pull shades about 1.5 times the vertices of an indexed draw on these meshes, so an indexed pull could save at most a third of that, about 0.1 ms. Far below the gate. Decision: not built, the plan's one draw path stands. The question can come back with authored 2k to 3k triangle L0 meshes, and the same measurement answers it.

### The contaminated runs

Three GPU runs between the good ones sat at exactly 60 fps with frames of 11.6 and 21.9 ms. During them three Blender python processes of the owner's model generation track used about ten cores (load average 5.5 to 6). The owner confirmed the timing. Three repeats after they finished read 101 to 155 fps. Rule from now on: log `/proc/loadavg` next to every measurement, and never quote a number from a run without it. work/scripts/gpu-pass-measure.sh prints it now.

### A hazard worth knowing: the pipelined handoff steals sim work

Bevy's pipelined renderer makes the main thread wait for the render world, and the render thread wait for the main world, inside `ComputeTaskPool` scopes that tick the shared executor while waiting. Bevy's own doc on that scope form: pulling tasks from the global executor "can run tasks unrelated to the scope and delay when the scope returns". The sim job's chunk tasks live in that executor. A thread that reaches the handoff while the job is queued gets conscripted into sim chunks until its own wake-up task comes around. On the CPU path the main thread arrived late and rarely waited. On the GPU path it arrives early. This shows up as the fixed loop shrinking (the job finishes sooner, the install waits less) and as a longer wait leg on frames where the job is still queued. The total work is conserved, so it is not a loss, but it is why the frame legs move around between runs. The honest mitigation named in devlogs 0079 and 0081 still stands: the sim job spawning fewer, larger tasks so the queue is empty when frame threads look at it. Not built here.

## State

- MERGED as PR #7 on 2026-09-22, `main` = a197872. Eight commits: the validation fix, the GPU path, the render thread timer, the bodies, the readback and check mode, the camera sweep knob, the default flip, a field trim. Build and clippy clean at every commit. Handoff for the next agent: work/handoffs/HANDOFF-after-item2-2026-09-22.md.
- The owner's feel pass was positive (above), so the GPU path is the default and `FL_GPU_SYNC=0` is the fallback. Archery on the GPU path: fingerprints equal the baseline 19 of 19, 11,700 check frames with no difference. Ready for the PR draft in work/drafts/pr-gpu-render-data.md.
- The 40 byte record (yaw pair and color packed) waits for the feel pass.
- Backup of the branch before a history fix of two broken commits: ref `refs/backup/gpu-render-data-pre-fix-20260922`, bundle work/backups/gpu-render-data-pre-fix-20260922.bundle, verified. The two commits had been made from zero-context patch splits that landed lines in the wrong places. They were replaced by commits made from exact file states, each checked to compile.

## Commands

```
FL_GPU_SYNC=1 cargo run --profile opt-dev                       # the GPU path
FL_GPU_SYNC=1 FL_GPU_CHECK=1 cargo run --profile opt-dev        # both paths, counts compared per frame
FL_GPU_SYNC=1 FL_LOD_DEBUG=1 cargo run --profile opt-dev        # tint by level, now in the vertex shader
WGPU_VALIDATION_INDIRECT_CALL=1 FL_GPU_SYNC=1 cargo run --profile opt-dev   # the slow validation back on
FL_GPU_SYNC=1 work/scripts/hashrun.sh dir-gpu 40 FL_TEST_DIR=1   # render only, hashes must equal the baseline
```
