# 0048 — perf/spikes: the fat tick debugged — preemption, not work

**Date:** 2026-07-21
**Branch:** `perf/spikes`, commit d7596b3 (chunk attribution)
**Owner ruling driving this:** no async-sim architecture; debug the fat
tick and remove the bottleneck. (His multiplayer/determinism instinct:
decoupled fixed-tick + interpolated render is in fact the standard
lockstep-RTS model and determinism lives in the fixed tick, not the
thread — but the deeper point stood: hiding an unexplained 3-6x tick
under interpolation would have buried a real bottleneck. He was right.)

## Method

Per-chunk attribution inside the integrate scope (d7596b3): every task
logs (wall ms, neighbor-candidates visited); `[spike-chunks]` on fat
ticks, `[chunk-base]` every 150 ticks as reference. Three signatures
separate: work blowup (sum+candidates up together), imbalance (one
chunk dominates), contention (sum up, candidates flat).

## Verdict (100k tour runs)

- March baseline: step 2.2-2.9 ms, 1.5-1.6 M candidates/tick.
- Steady melee: step 4-7 ms, 2.6-3.0 M candidates. The crush costs ~2x
  work — comfortably inside the 33 ms budget. NOT the spike.
- Spike ticks: step 14-19.5 ms at 2.0-2.2 M candidates — **flat work,
  2.5x summed chunk time, up to 5.7x wall vs an adjacent tick with the
  same candidate count.** The parallel scope's workers are preempted
  mid-task (render-world CPU bursts, driver threads, OS timeslice); a
  descheduled worker holds its chunk hostage and the scope waits.
  Confirms 0034's 200k conclusion at 100k. Chunk size cannot fix this
  (a stolen core stalls whatever task it holds); the fix is fewer
  competing runnable threads during the tick.

## FL_THREADS sweep (quiet box, tour, ~150 s runs)

| config | spike ticks | step p50 | p90 | max |
|--------|-------------|----------|-----|-----|
| 100k default (24) | 220 | 4.7 | 9.5 | 19.5 |
| 100k FL_THREADS=22 | 111 | 4.6 | 6.4 | 11.0 |
| 100k FL_THREADS=20 | 120 | 4.5 | 6.8 | 13.0 |
| 100k FL_THREADS=16 | 108 | 5.2 | 5.8 | 7.8 |
| 200k default | 276 | 6.3 | 8.3 | 14.6 |
| 200k FL_THREADS=20 | 285 | 7.0 | 8.5 | 10.3 |

At 100k, reserving 2-4 cores halves the spike count and tail at zero
mean cost (sweet spot 20-22; 16 flattens ticks further but mean starts
rising). At 200k the mean regresses ~11% (throughput-bound, as 0034
found) while the tail still improves — so the default stays full
width; **FL_THREADS=20 is the recommended setting for 100k recording
sessions on the 24-core box.** Frame-level hitch counts across runs
are render-burst-dominated and noisy; step percentiles are the honest
sim metric.

## What remains (the actual thieves)

1. **Render-world CPU bursts** — 14 ms render-thread CPU in slow
   frames vs 2.5 normal (trace, 0047): bevy batching/mesh
   preprocessing at far zoom. Shrinking this removes the preemptor.
   The old "camera-move dips" item, now the primary target.
2. Optional: melee candidate reduction (per-cell team masks to skip
   same-team-only cells beyond separation radius) — halves steady
   melee cost, straightforward, mechanics-identical, but not the
   spike cause.
3. One-time startup warmup frames (100-190 ms) — trim recordings.

## Recording advice (today, no further work)

`FL_UNITS=50000 FL_THREADS=20 cargo run --profile opt-dev` (+FL_MAP,
FL_VOLUME to taste). Worst observed tick 13 ms → worst frame ~20 ms ≈
one dropped frame at 60 fps; catch-up clamp caps anything pathological.
