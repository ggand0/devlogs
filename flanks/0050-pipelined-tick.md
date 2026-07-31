# 0050 — perf/spikes: the tick leaves the frame path (the fix)

**Date:** 2026-07-21
**Branch:** `perf/spikes`, commit d64e6d4
**Owner mandate:** "fix it completely and fundamentally. no lazy hacks."

## What his session log proved (tee'd /tmp/fl.log, 100k, FL_THREADS=20)

53 hitch frames in ~90 s: ~40 were frames carrying a contention-
inflated sim tick (tick totals 12-27.6 ms, step to 23 ms), ~8 were
Update-side stalls, 0 catch-up bursts. His selection/attack-order
correlation did not survive his own re-test and was dropped. The
dominant class was structural: the tick ran inside the render frame.

## The fix

The kinematic tick (grid rebuild + integrate) now runs as a fully
self-contained job on a **dedicated worker thread** between fixed
ticks. FixedUpdate does: install finished job (buffer swaps, ~0 cost)
→ serial apply (damage/deaths/morale/orders — code unchanged, same
order) → kick the next job after order clearing (so the job reads
exactly the state the old inline prep read). The renderer keeps
interpolating the last two completed states, as it always has. A
contention-inflated tick now delays its own completion inside the
33 ms budget instead of stretching a frame.

- `TickJob` owns input copies (~5 MB/tick, buffers recycled — steady
  state allocates nothing). Per-regiment command snapshot at kick =
  one-tick input quantization (the lockstep model).
- Terrain rides an `Arc` snapshot (`SharedTerrain`), refreshed on
  change. NOTE: if crater carving ever returns, the job sees terrain
  edits one tick late.
- `Units.generation` guards restarts: an in-flight job from a dead
  world is detected stale and dropped (R works mid-battle).
- `FL_PIPELINE=0` = inline fallback, bit-identical to the old sim.

## The deadlock that almost shipped

First version ran the job on `AsyncComputeTaskPool` and hung ~1/3 of
FL_TEST_ROUT startups. gdb backtraces of a live hang: `step_sim`
(running as a system task) parked in `block_on`, while the job it
waited for sat queued on a pool whose workers can themselves pick up
system tasks and park — executor priority inversion; occasionally
every thread allowed to start the job was wedged. Fixes: (1) the job
runs on a plain OS thread (always runnable, mpsc channels, a panic in
the job surfaces as a crash instead of a silent hang), (2) the job's
scopes use `scope_with_executor(false, ..)` so the waiting worker
thread never pulls a parked system off the shared executor and closes
the cycle from the other side — chunk tasks still fan out across all
pool workers.

## Verification ladder (all green)

1. **Refactor soundness:** FL_PIPELINE=0 reproduces the pre-refactor
   FL_HASH fingerprints 19/19 — the restructure changed zero math.
2. **Determinism:** pipelined mode run-vs-run 19/19 hashes equal.
3. **Bands:** DIR final sector tallies IDENTICAL to baseline
   (front 475 / side 349 / rear 568, dmg/hit 22.4/29.2/49.0); CHARGE
   wall-holds-open-bowls ordering intact; ROUT breaks at 53% (band
   ~50%) and rallies; SURROUND pocket 0 / line 234 — identical to
   inline mode; FORM all OK (nn wall 1.06 < normal 1.36 < loose 1.89).
4. **Stability:** 6/6 clean FL_TEST_ROUT runs (the deadlock repro).
5. **Perf, 150 s 100k tour:** hitch frames 165 → 40 (−76%), and the
   tick-carrying class is GONE — every remaining hitch is render-side
   (main 25-29 ms, tick totals 2.6-6.7 ms). The tick itself flattened
   off the frame path: step wall p50 3.7 / p90 4.6 / max 7.2 ms
   (pre-pipeline: 4.7 / 6.8-9.5 / 13-19.5). FL_THREADS tuning is no
   longer needed at 100k.
6. **200k:** tick totals 6-9 ms in hitch frames; the remaining 200k
   hitches are far-zoom render throughput (main 6-8 ms, frame 32-48 —
   render back-pressure), i.e. the pre-existing render backlog item.

## What remains (the render class)

~40 tour hitches / 150 s at 100k are render-CPU bursts at far zoom
(25-29 ms ≈ one dropped frame at 60 fps). That's branch item 2
(batching/GPU-preprocessing cost), unchanged by this work and now the
only spike source left. The [hitch] log now cleanly separates it:
`main` high + `tick total` low = render; anything else would be new.

## For the recording

`FL_UNITS=50000 cargo run --profile opt-dev` — no FL_THREADS, no
FL_CATCHUP tuning needed anymore. Trim the first second (shader
warmup frames). `[spike]` lines still appear in logs — they now
measure background job wall time (diagnostic), not frame events.
