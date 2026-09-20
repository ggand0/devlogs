# 0079: Item 0, the sim tick leaves the frame path on `main` (2026-09-20)

Branch `perf/pipelined-tick`, from `main` e80dbe8. Plan: docs/plans/scale-to-1m.md, item 0. Design origin: devlog 0050 on the shelved `perf/spikes` branch. This is a re-port onto today's sim, which has since gained archery, per-soldier arrow aim, morale and fatigue.

## Commits

- e3f6aeb: `FL_HASH` state fingerprint.
- 36dd2cd: the tick restructured into a self-contained job, still computed inline. Behavior unchanged, proven by fingerprints.
- 18ada02: the job runs on a dedicated worker thread. `FL_PIPELINE=0` keeps the inline path.
- Catch-up clamp (`FL_CATCHUP`) with the frame pacing log, then `FL_SIM_POOL`, as the last two commits on the branch.

## FL_HASH

FNV-1a over position x and z, facing, hit points, death timer, swing state, swing timer and ammo. Logged every 150 ticks, or every n ticks with `FL_HASH=n`. Wider than the July version, which skipped facing, swing and ammo.

First finding: today's sim is bit-deterministic run to run, archery and AI included. Three scenarios, 19 fingerprints each, 40 seconds, equal 19 of 19:
- `FL_TEST_ARCHERY=1` (2.9k soldiers)
- `FL_TEST_DIR=1` (2.2k soldiers)
- `FL_TEST_FRONT=1 FL_UNITS=20000` with the AI on (40k soldiers, 8.5k dead by the end)

Baselines from the code before any restructure are in tmp/hash-baselines/*-e80dbe8.hash. Scripts: tmp/scripts/hashrun.sh and tmp/scripts/hashcmp.sh.

## The restructure

`step_sim` was one 1,470-line function: prep, parallel integrate, serial apply. It is now three parts.

- `prepare_tick` fills a `TickJob` on the main thread. Column copies are recycled buffers, about 59 bytes per soldier, 12 MB per tick at 200k. It also resolves orders, the per-regiment snapshot and the archers' fire solutions (`ShootAt`, the living members of targeted regiments, the friendly blocks arrows must loft over). It makes the two regiment writes prep always made: the stand-off anchor snap and the `firing` flag.
- `run_tick_job` rebuilds the grid and runs the integrate on the job's own data. No ECS access. The integrate body is untouched, only its binding preamble changed.
- `step_sim` installs the finished job with buffer swaps: positions, the nine read-modify-write columns (ammo is new since July), the grid, the damage event buffers and the arrow spawn buffers. Then the serial apply runs as before.

New since the July design, and handled:
- Arrow spawns are an output of the job. They swap into `ArrowSpawns` at install and `update_arrows` drains them the same tick, as before.
- `update_arrows` writes soldier columns (hp, flash, death). It runs before the kick, like every other post-step system, so no write can be overwritten by an install. The kick is ordered after `clear_arrived_orders`, and every post-step system is ordered before that.
- `apply_reforms` writes only `home`, which the job reads but never swaps back. One tick of staleness, no lost write.
- Player input in `Update` writes soldier columns only during deployment, when the sim is frozen and no job is out.
- `Units.generation` is bumped in `setup_battle`. A job from the previous battle is dropped at the next install.
- `Terrain` became `Clone`. The job reads an `Arc` snapshot that refreshes when the resource changes.

## The hang, and its real cause

The first pipelined build froze at battle start about one launch in two. The owner saw it as black windows. My first guess was tasks stranded in a blocked pool thread's local queue. That was WRONG. gdb settled it in one run.

The backtrace of the sim worker thread: `run_tick_job` -> `SpatialGrid::rebuild` -> bevy scope -> `block_on` -> a bevy system with 12 parameters -> channel `recv`. The worker thread was running `step_sim`. While waiting inside the grid rebuild's scope it ticked the shared executor, stole the `step_sim` system task, and that system then parked waiting for the job its own thread was computing.

Devlog 0050 describes exactly this cycle and fixed it for the integrate scope. In July the grid rebuild had no scope of its own. Today it has two, and I had protected only the integrate.

The fix belongs to the thread, not to call sites. `util::sim_scope` is the one scope every sim kernel uses. On the worker thread (marked by a thread local at startup) it waits without ticking the shared executor. Everywhere else it is a plain scope. `take_in_flight` asserts it is not on the worker thread, so a repeat crashes with a message instead of hanging.

After the fix: 10 of 10 short launches clean (DIR and ROUT alternating), plus every later run.

How to take the backtrace without ptrace rights on a running process: start the game under gdb and let `timeout -s INT` interrupt it.

```
timeout -s INT 22 gdb -q -batch -ex run -ex "thread apply all bt 14" --args ./target/opt-dev/flanks
```

## Verification

| Check | Result |
|---|---|
| `FL_PIPELINE=0` against the pre-restructure baseline | 19 of 19 equal in all three scenarios, checked after the restructure and again after the hang fix |
| Pipelined, run against run | 19 of 19 equal in all three scenarios |
| Pipelined against inline | differs, as designed. Commands quantize to the tick. Alive at tick 1140: archery 2879 against 2867, DIR 2153 against 2153, front 31507 against 31538 |
| Startup stability | 10 of 10 |
| DIR battery, pipelined, 110 s | front 586 / side 297 / rear 616, dmg per hit 22.1 / 28.5 / 49.6. IDENTICAL to the inline baseline |
| CHARGE battery, pipelined | in band. Wall lane holds: spears 143 at dz -5.4 m, heavies 108. Open lane spears run down 360 m and cut to 27 |
| ARCHERY battery, pipelined against an inline run of the same build | middle block at t=24/40/56/72/88 s: 477/363/135/2/0 pipelined, 474/377/128/3/0 inline. Same pace. Devlog 0075's "wiped by about t=45 s" was loose, it takes about 75 s in both modes |
| `FL_SIM_POOL=12` and the clamp | front battle fingerprints equal the earlier pipelined run 19 of 19 |

## Frame pacing at 200k (the point of the item)

Locked 900 m view, culling off, 197k soldiers. Instrument added to the periodic log: frame intervals split by whether the frame carried a tick, and the wall time between `FixedFirst` and `FixedLast`. My first attempt used Bevy's real-time delta and got the labels swapped, because that delta lags a frame under pipelined rendering. The instrument now uses its own clock.

| | Inline | Pipelined |
|---|---|---|
| Fixed tick holds the frame | 15.2 ms | 6.3 ms |
| Frame with a tick | 25.5 ms | 17.7 ms |
| Frame without a tick | 9.5 ms | 15.3 ms |

Reading:
- The alternating 9.5 and 25.5 ms pattern became a nearly even 15 and 18 ms. The worst regular frame is 8 ms shorter. That is the goal of item 0.
- Frames without a tick got slower. The job's chunk tasks run on bevy's compute pool, where they queue in front of the frame's own parallel work (instance sync, render prep). The tick still costs the same CPU. It now overlaps the frame instead of blocking it.
- The 6.3 ms left inside the fixed tick is the serial apply, the field, group updates, arrows, morale, deaths, and the 12 MB of column copies for the next job.
- The box carried other CPU load during every run (load average 6 to 9). Absolute numbers are pessimistic. Rerun on a quiet machine before quoting them.

`FL_SIM_POOL=n` gives the worker its own n threads instead of bevy's pool. Measured at 8, 12 and 16: frames without a tick drop to about 12.5 ms, frames with a tick stay near 17 ms. A modest gain, within today's noise between sizes. Default stays off until it is measured on a quiet box. Next idea if pacing needs more: run the sim threads at lower OS priority, so frame work always wins a core. That needs the `libc` crate.

## Catch-up clamp

Ported from 1a62029. `Time<Virtual>` max delta is `FL_CATCHUP` tick periods, default 2. Bevy's default 250 ms allows 7 back-to-back ticks after one slow frame, and in pipelined mode every tick after the first in a frame waits for its job in full. Time past the clamp is dropped, so a stall plays as a moment of slow motion. `[catchup] n sim ticks in one frame` logs any burst.

## Commands

```
cargo run --profile opt-dev                      # pipelined (default)
FL_PIPELINE=0 cargo run --profile opt-dev        # inline fallback, bit-identical to before
FL_SIM_POOL=12 cargo run --profile opt-dev       # sim job on its own 12 threads
FL_HASH=60 FL_TEST_DIR=1 cargo run --profile opt-dev
tmp/scripts/hashrun.sh name 40 FL_TEST_DIR=1 && tmp/scripts/hashcmp.sh a b
```

## Open

- OWNER FEEL PASS. Orders now land one tick (33 ms) later than before. It was accepted in July, but he has not played this build.
- A mid-battle restart while a job is in flight is covered by the generation check in code. It was not exercised by a scripted test.
- Not ported from `perf/spikes`: the `[hitch]` whole-tick attribution (800cece) and per-chunk spike attribution (d7596b3). The pacing log covers the need for now.
- Column copies cost about 12 MB per tick at 200k on the main thread. Item 2 (GPU render data) will want the job's buffers as its upload source, which is the moment to revisit what is copied.
