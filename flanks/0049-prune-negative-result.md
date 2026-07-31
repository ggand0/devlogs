# 0049 — perf/spikes: scan prune negative result; determinism proven; SMT

**Date:** 2026-07-21
**Branch:** `perf/spikes`, commit 7a2697b (FL_HASH)
**Owner data driving this:** spikes persist at FL_THREADS=20; the
selection/attack-order correlation he saw did NOT hold up (his follow-up:
spikes occur without input too — the input path was a coincidence and its
investigation was dropped).

## SMT context (box facts)

The box is a Ryzen 9 3900X: 12 physical cores, 24 SMT threads. Pool
sweep including physical-core width: FL_THREADS=12 gives the flattest
ticks yet at 100k (max 8.6 ms) but +23% mean; at 200k it's clearly
throughput-bound (p50 6.3→10.5 ms, spike lines 276→1335). Confirms the
"2.5x wall on flat work" signature as SMT-sibling + timeslice
contention, and that pool sizing is a mean-vs-tail dial, not a fix.
FL_THREADS=20 stays the 100k recording recommendation.

## Cell-prune experiment: built, proven, measured at zero, REVERTED

Idea (mechanics-identical by construction): per-cell team-presence bits
in the grid; the fused scan skips whole cells that are same-team-only
AND beyond SEP_RADIUS (such candidates fail every visitor guard —
targeting/impale need an enemy, same-team physics caps at 1.4 m); the
4 m wide-acquisition (enemy-only visitor) skips ALL same-team cells.

- Correctness: PROVEN digit-identical via FL_HASH (below) — 19/19
  state hashes equal, prune on vs off, over an FL_TEST_DIR battle.
- Payoff: candidates/tick 2.44M → 1.31M (−46%) at 100k tour… and step
  time UNCHANGED (p50 4.9 both, chunk-sum ~60 ms both, max flat).
  A far same-team candidate costs a few ns to reject — the scan volume
  was never where the time lives. Per-unit fixed work (steering,
  terrain sampling, swing FSM, memory traffic) dominates the tick.
- Ruling per the 0020 precedent: measured-zero optimizations don't
  ship. Reverted from spatial.rs/movement.rs; this entry is the record.
  Don't rebuild it without first shrinking the per-unit fixed work.

## Keeper: FL_HASH, and a method correction

`FL_HASH=1` logs an FNV-1a fingerprint of every unit's position + hp +
death timer every 150 ticks. Findings:

1. **The sim is bit-deterministic tick-for-tick** (two identical runs:
   19/19 hashes equal). Parallel chunks read only last-tick state and
   events apply serially in chunk order, as designed. Good news for
   replays/multiplayer.
2. **The old "digit-identical" log-diff method is unsound**: the
   FL_TEST samplers ride wall time, so the SAME binary prints
   different casualty digits run to run (verified: base-vs-base DIR
   logs differ at t=12s). Past log-diff-based identity claims were
   band comparisons in practice. FL_HASH is the gate from now on.

## Where the branch stands

- Landed: catch-up clamp (0046), attribution instrumentation
  (0047-48), FL_HASH (0049). Root cause: core contention/preemption,
  SMT-amplified; crush work itself fits the budget.
- Recording recipe: `FL_UNITS=50000 FL_THREADS=20 FL_CATCHUP=1`
  (+`2>&1 | tee /tmp/fl.log` so a played session's [hitch]/[catchup]
  lines survive for attribution). FL_CATCHUP=1 keeps any slow frame to
  ONE tick (~13 ms worst at t20) at the price of rare millisecond-scale
  slow-mo — for a video that trade is free.
- Next fundamental work, in order: (1) shrink render-world CPU bursts
  (bevy batching/prep at far zoom — the preemptor; 14 ms in slow
  frames), (2) shrink per-unit fixed tick cost (terrain sampling is
  4-6 lookups/unit/tick — cache candidates), (3) startup warmup
  (cosmetic; trim recordings).
- 12 GB `trace-1784607636205616.json` still in repo root — owner's
  call to delete.
