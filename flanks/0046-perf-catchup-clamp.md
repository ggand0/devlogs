# 0046 — perf/spikes: fixed-timestep catch-up clamp

**Date:** 2026-07-21
**Branch:** `perf/spikes` off main 31d7435 (map-art merge), commit 1a62029
**Goal:** recordable gameplay without lag spikes (owner wants to post a video on X)

## Context

Devlog 0034 round 5 closed the melee perf gate but left the lag spikes
unfixed, with the corrected diagnosis: the render app runs pipelined on
threads concurrent with the sim tick, so render prep (instance sync,
CPU culling, driver threads) steals cores; one frame overruns 33 ms and
the fixed timestep then runs multiple catch-up sim ticks in a single
frame, which overruns *that* frame — the classic spiral. Owner directive
was a dedicated perf branch with scope: (1) catch-up clamp, (2) render
prep / GPU culling, (3) verify on PLAYED sessions via the `[spike]` log.

New owner data point this session: **spikes happen at 100k too, where
everything is otherwise smooth.** That confirms the spiral framing —
the trigger is frame *variance*, not mean throughput. At 100k the burst
is the whole problem, so item (1) alone should make the recording clean.

## The change (one focused edit + instrumentation)

- `main.rs`: Bevy's `Time<Virtual>` default `max_delta` is 250 ms — an
  overrun frame may be followed by up to ~7 back-to-back catch-up ticks
  at 30 Hz. Now inserted as
  `Time::<Virtual>::from_max_delta(FL_CATCHUP / 30 s)`, **FL_CATCHUP
  default 2** (min 1). Time beyond the clamp is dropped: a stall plays
  as a momentary imperceptible slow-mo instead of a tick burst.
- `overlay.rs`: `TicksThisFrame` counter (FixedUpdate increments,
  Update reports + resets) logs `[catchup] N sim ticks in one frame`
  whenever N > 1. This is the verification signal: bursts above
  clamp + 1 would mean the clamp isn't holding.

## Evidence (fixed-camera 45 s runs, 200k, this box)

| run | [catchup] events | burst sizes |
|-----|------------------|-------------|
| FL_CATCHUP=8 (≈ old behavior) | 10 | 9× 2-tick, **1× 6-tick** |
| default (clamp 2) | 5 | 5× 2-tick, nothing above |

The 6-tick burst under the loosened clamp is the spike phenomenon
occurring naturally and getting through; the default clamp caps every
frame at 2 ticks. Small-scale sanity run clean (no panics, battle
proceeds, fps normal).

Known limitation of these runs: fixed camera — per the 0034 directive
they don't reproduce the camera-motion spikes. **Owner play-test
protocol:** play normally (camera motion included) at 100k, watch for
felt hitches and grep the log for `[catchup]` / `[spike]`. Expect
`[catchup] 2` occasionally and nothing higher; a hitch that still
*feels* bad with no big catchup burst is render-side (item 2 backlog).

## Cost / semantics

- Sim time lost past the clamp is dropped, not repaid — battle clock
  drifts slightly behind wall clock across a stall. Irrelevant here.
- Slow-mo only engages when a frame exceeds FL_CATCHUP × 33.3 ms
  (default: below ~15 fps momentarily). One such frame is invisible;
  the old alternative was a visible freeze.

## Remaining branch scope

- Render prep cost / GPU culling (camera-move dips at 200k) — probably
  unnecessary for the 100k recording; measure after the owner's pass.
- Dead ends on record, do not retry: FL_THREADS pool narrowing
  (negative, 0034), SIMD scan kernel without AoSoA (0020).
