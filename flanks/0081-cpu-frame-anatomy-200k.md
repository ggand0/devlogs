# 0081: Where the CPU frame goes at 200k (2026-09-22)

Branch `perf/pipelined-tick`, measured after item 0 (devlog 0079). Purpose: a reference for the item 2 design in docs/plans/010-scale-to-1m.md. Every number here is from the dev box (Ryzen 9 3900X, 12 cores, 24 threads, RTX 3090), 197k soldiers alive, the locked 900 m view with culling off, muted, `FL_TEST_FRONT=1 FL_UNITS=100000 FL_AI=0 FL_ENEMY_STATIC=1`.

## The instrument

Two lines in the periodic log, both added on this branch:

- `frame ms without a tick ... | with a tick ... | inside the fixed tick ...`: frame intervals split by whether the frame ran a sim tick, and the wall time between `FixedFirst` and `FixedLast`. Own `Instant` clock. Bevy's real-time delta lags a frame under pipelined rendering and swapped the labels on the first try.
- `main thread ms: fixed loop | update+post | extract+wait render || instance copies: extract | write_buffer`: three marks on the main thread (First, end of the fixed loop, Last) plus timers on the two copies of the instance data. `EXTRACT_US` and `PREPARE_US` in render_units.rs carry the copy times across threads.

## The frame, new mode

```
MAIN THREAD, one frame                                            ms
  input and window events                                        0.3
  every other frame: the sim tick                                6.4
     take the finished job from the worker (normally no wait),
     swap its buffers in, damage apply, arrows, deaths, fatigue,
     morale, order clearing, copy 12 MB of columns, kick next job
  build render data for every soldier (parallel, all cores)      4.7
  everything else: camera, orders, cards, overlay, banners       6.6   (2.4 when the sim job is not competing)
  copy render data into the render world (main thread blocked)   2.0
  wait for the render thread to release the previous frame       0.5
                                                        frame:  16 to 17

RENDER THREAD, overlapping the next main frame
  write_buffer: copy render data into GPU staging                1.5
  batching, command encoding, submit                             small, unmeasured
  then the GPU works                                             4.0

WORKER THREAD, overlapping everything above, one job per tick
  grid rebuild 4.7 + integrate 7                                12
```

Old mode for comparison: no worker thread. The whole 15 ms tick sat inside the frame that carried it, so frames alternated between 9 and 25 ms. In the new mode every frame pays about 4 ms of contention instead, and frames run an even 16 to 17 ms. On a clean desktop the averages were 57 fps old, 66 fps new.

## The legs, measured

Medians over one two-second log period. This run carried a little more load than the clean rerun in 0079, the split is what matters.

| Leg | Inline | Pipelined | What is in it |
|---|---|---|---|
| Fixed loop, on frames with a tick | 15.0 | 6.4 | Inline: grid 2.8, integrate 6.3, field 0.7, the apply and the post-step systems. Pipelined: install swaps (about 0), field 0.8, damage apply, arrows, deaths, fatigue, morale, orders, then the kick's 12 MB of column copies |
| Update and PostUpdate | 6.6 | 11.3 | Instance sync 4.2 against 4.7. Everything else 2.4 against 6.6 |
| Extract and wait for the render thread | 2.4 | 2.4 | Extract memcpy of the 16 instance buckets 1.9 to 2.0. Wait about 0.5 |
| Render thread, `write_buffer` | 1.4 | 1.5 | Overlaps the next main frame |

Cross-check, inline: a frame without a tick is 0.3 + 6.6 + 2.4 = 9.3 ms, measured 9.3. A frame with a tick adds 15, measured 24.8.

## What the instance sync does, per soldier per frame

`sync_instance_data` in render_units.rs, parallel over 16k-soldier chunks: interpolate position and facing between the last two ticks by the fixed clock's overstep, frustum cull against a fresh camera frustum, pick a detail level with jitter and hysteresis, run the four pose smoothers (walk, stance band, march, wall), write a 64 byte `InstanceData` record into one of 16 kind-by-level buckets, then concatenate the chunk outputs. Corpses go through the same pass. 12.6 MB of output at 197k. That output is then copied twice more: once into the render world at the sync point (extract, 2.0 ms), once into GPU staging (`write_buffer`, 1.5 ms).

## Bevy's compute pool, and why the sim job slows every system

Bevy splits the 24 hardware threads by its default policy: 4 threads for file loading (`IoTaskPool`), 4 for background jobs (`AsyncComputeTaskPool`), and the remaining 16 for the `ComputeTaskPool`. Nearly everything runs on the compute pool:

- Every Bevy system. The multi-threaded schedule executor runs each system as a task on this pool (bevy_ecs `multi_threaded.rs`), in the main world and in the render world alike.
- Bevy's own parallel work: transform propagation for the banner, vegetation and terrain entities, visibility checks, mesh and material batching on the render side, shader pipeline compilation.
- Our own parallel loops: the instance sync, the frontline density field, the neighbor audit, and now the sim job's grid rebuild and integrate.
- Not on it: asset loading, audio (its own thread), the window event loop.

Consequence: when the sim job fills the pool with about 100 chunk tasks, every Update system waits in the queue behind them, not only the parallel sync. That is the 4.2 ms swell on "everything else" (2.4 to 6.6 ms). The sync itself grows only 0.5 ms.

### FL_SIM_POOL, tried and dropped

A second pool of 12 threads reserved for the sim job (`FL_SIM_POOL=12`) brought frames without a tick from 15.1 to 11.6 ms, a consistent 1 ms per frame on average, 66 to 71 fps. The owner rejected it as a default and it was dropped from the branch (devlog 0079 has the details and the backup location). His reading was right: 16 pool threads plus 12 sim threads is 28 busy threads on 24 hardware threads. It works through OS scheduler fairness, taking cores from Bevy's workers, and only because the 8 file and background threads sit idle. On an 8-thread laptop the same trick can go the other way. If headroom is ever needed here, the honest form stays inside the one pool: the sim job spawns fewer, larger tasks so a few workers remain free for systems.

## What items 2, 6 and 7 change in the picture

| Line | Today | Item 2, render data built on the GPU | Items 6 and 7, less sim work |
|---|---|---|---|
| Build render data for every soldier | 4.7 | Gone from the CPU. A compute shader does it from the sim state, and the GPU has about 11 ms idle per frame | |
| Copy render data into the render world | 2.0 | About 0. The upload source becomes the tick's compact state, about 40 bytes per soldier once per tick instead of 64 bytes per frame, handed over by swapping buffers | |
| `write_buffer` on the render thread | 1.5 | About 1 per tick, and the render thread was never the limiter | |
| Everything else | 6.6 | Toward 2.4, only partly. With no second heavy job on the pool the sim job finishes in about 8 ms instead of 12 and overlaps fewer systems | Further toward 2.4. A shorter sim job holds the pool for less of the frame |
| Sim tick, every other frame | 6.4 | Unchanged | Unchanged. This is the serial apply plus the 12 MB copy into the next job |
| Worker thread | 12 | About 8 | Item 6 targets 1.5x less integrate work. Item 7 lets standing soldiers skip their step (deferred by the owner) |

Net: item 2 alone takes the frame from 16 to 17 ms to roughly 8 to 10 ms at 200k. The saving grows with the army, because the render data cost stops scaling with soldier count on the CPU. Items 6 and 7 do not touch the frame directly. They shrink the sim job, which shortens how long it competes with the frame, and push out the army size at which the tick stops fitting its 33 ms budget.

## What stays after all three

- The 6.4 ms serial tick still lands in every other frame. About 1.5 ms of it is the column copy into the next job, which double-buffered columns could remove. The rest is the serial apply, the field and the regiment systems. Fine at 200k, and it grows with events per tick.
- Any parallel job on Bevy's compute pool delays systems, by design of the executor. Bevy's pool has no priorities.
- The render thread is not the limiter in this view. The main thread waits about 0.5 ms for it. Devlog 0047 traced 14 ms render-thread bursts at far zoom on an older Bevy, none were seen here.

## Notes for the item 2 design

- The per-tick upload is the natural handoff point: the job's output columns already hold exactly the state the compute shader needs. Swap, do not copy.
- The GPU has room. In this view it works about 4 ms of a 15 ms frame at full clock.
- Measure before and after with the two log lines above, in the four locked views of devlog 0077. The `main thread ms` line is the pass criterion for "render CPU cost stops growing with army size".
