# 0047 — perf/spikes: the spikes attributed (crush ticks, not render)

**Date:** 2026-07-21
**Branch:** `perf/spikes`, commits 1a62029 (clamp, devlog 0046) + 800cece (instrumentation)
**Owner report driving this:** with the clamp in, a 100k tour run still showed
9-10 felt spikes, roughly every 10-12 s, **even with the camera on empty ground**.

## Instrumentation added (800cece)

- `[hitch]` log: any real frame > FL_HITCH_MS (25) prints main-world span
  (First→Last), ticks run that frame, whole-FixedUpdate wall time
  (`SimStats.total_ms` — process_deaths/morale/orders were previously
  untimed), per-phase sim ms, and sync ms.
- `FL_CAM_TOUR=1`: scripted orbit + pan + zoom sweep for spike hunts.
- `FL_NO_AUDIO=1`: skips audio entirely (FL_VOLUME=0 leaves all five
  beds decoding at zero volume — now provably irrelevant to the spikes).

## Evidence chain (100k = FL_UNITS=50000, tour camera, this box)

1. Frame time does NOT track drawn count: 3k drawn rendered at 16-24 ms
   during the march phase of one run while 49k drawn rendered at 5.5 ms
   post-contact. GPU pass timings stayed ~1.4 ms throughout. Whatever
   spikes, it isn't draw volume. (That one run's continuous march-phase
   slowness never reproduced across four later runs — external box
   contention, not signal.)
2. `[hitch]` attribution over the battle phase shows the repeating spike
   class: frames at 25-37 ms whose whole-FixedUpdate is 18-30 ms with
   `step` 13-24 ms (baseline 3-5 ms). Episodes recur every ~9-18 s —
   the owner's felt cadence — and are camera-independent.
3. Chrome trace (bevy `trace` + `trace_chrome`, 80 s run, 9093 frames,
   153 slow >25 ms): slow frames average 36 ms of main-path time, of
   which FixedUpdate self 21.5 + integrate 13.6 + grid 3.3 ms — the sim
   tick dominates. Concurrent render-thread CPU is 14.2 ms in slow
   frames vs 2.5 in fast (contention amplifier, secondary). Worker
   busy-time shows ~23 workers mostly idle during the stall — but our
   par chunks are unspanned, so busy-vs-starved is settled instead by
   devlog 0038's fixed-camera data: crush ticks cost 25 ms at 200k with
   NO camera motion. The crush cost is real algorithmic work.

**Verdict: the periodic felt spike is a dense-melee ("crush") sim tick
landing on the render frame's critical path.** Charge collisions and
melee pockets balloon the integrate phase 3-6x for a few ticks; each
such tick turns a ~5 ms frame into 25-40 ms. The 0046 clamp already
prevents burst amplification (two such ticks in one frame max, worst
~50 ms); it cannot help with a single expensive tick. Render prep
contention (item 2 of the branch) inflates but does not cause it.
Startup also shows two one-time 100-190 ms frames (pipeline/asset
warmup) — cosmetic for recording (trim the first second).

## Fix directions (owner ruling wanted)

A. **Take the tick off the frame path** — run the sim tick in a
   background task (AsyncComputeTaskPool); the renderer keeps its own
   pace and interpolates between the last two completed sim states
   (sync_instance_data already interpolates via overstep_fraction, and
   yaw_prev/pos_prev exist). A 25 ms crush tick then fits the 33 ms
   30 Hz budget without any frame feeling it — at 100k AND 200k. No
   mechanics change: same kernel, same system order, same data; the
   tick just completes across frame boundaries. Invasive: restructures
   the FixedUpdate chain (step → deaths → morale → orders apply on
   tick completion), input→order latency +≤1 frame. The fundamental
   fix, M2TW-class engines run exactly this split.
B. **Make crush ticks cheap** — attack integrate's dense-pocket cost.
   Honest work (profiling the kernel under crush, cache layout), but
   0020's SIMD lesson says no quick win without AoSoA, and anything
   like neighbor caps changes mechanics (owner veto territory).
C. **Live with it for the video** — at 60 fps recording a 30 ms frame
   is one dropped frame; the clamp caps the worst case. Record at 100k
   and the spikes may not read on video. Zero work, fixes nothing.

## Also seen (branch backlog)

- Render-side CPU at far zoom (14 ms class): bevy batcher/GPU
  preprocessing + our prep — the old "camera-move dips" item, now with
  numbers. Separate from the periodic spikes.
- `frontline::update_groups` 1.0 ms in slow frames vs 0.03 baseline —
  minor, same contention signature.

## Verification protocol unchanged

Play (or FL_CAM_TOUR) at 100k, grep `[hitch]`: spike class shows as
`tick total ≈ step ≫ baseline`. After fix A those lines must vanish
(frames stop carrying tick cost); `[catchup]` still guards the clamp.

## Note

`trace-1784607636205616.json` (12 GB, repo root) is the raw trace —
owner to delete when done (rm needs his say-so).

Moved 2026-09-26: the trace now lives only at
`/data/ggando/flanks/assets_dev/trace-1784607636205616.json` (verified
byte-identical with `cmp` before the local copy under `assets_dev/` was
removed). It is kept only for re-analysis of this run; it can be deleted
whenever the space is wanted.
