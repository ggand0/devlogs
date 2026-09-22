# 0079: Item 0, the sim tick leaves the frame path on `main` (2026-09-20)

Branch `perf/pipelined-tick`, from `main` e80dbe8. Plan: docs/plans/scale-to-1m.md, item 0. Design origin: devlog 0050 on the shelved `perf/spikes` branch. This is a re-port onto today's sim, which has since gained archery, per-soldier arrow aim, morale and fatigue.

## Commits

- e3f6aeb: `FL_HASH` state fingerprint.
- 36dd2cd: the tick restructured into a self-contained job, still computed inline. Behavior unchanged, proven by fingerprints.
- 18ada02: the job runs on a dedicated worker thread. `FL_PIPELINE=0` keeps the inline path.
- 62d89e6: catch-up clamp (`FL_CATCHUP`) and the frame pacing log.
- fd8cfe1: the fps readout shows a two second average.
- 08501cf: main thread frame breakdown in the periodic log (devlog 0081 reads it).
- 9ef6fa6: pacing samples reset on battle start, found in the final diff review (menu time leaked into the first samples).
- DROPPED on 2026-09-22 at the owner's call: `FL_SIM_POOL`, was 846a327. See "FL_SIM_POOL, built and dropped" below. Backups: tmp/backups/0001-Add-FL_SIM_POOL-to-run-the-sim-job-on-its-own-thread.patch, the bundle tmp/backups/pipelined-tick-pre-drop-20260922.bundle (verified), and the ref refs/backup/pipelined-tick-pre-drop-20260922. The two commits after it were rebased and got new hashes.

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

### FL_SIM_POOL, built and dropped

`FL_SIM_POOL=n` gave the worker its own pool of n threads instead of bevy's compute pool. Measured at 8, 12 and 16 on the loaded box, and at 12 on the clean desktop: frames without a tick dropped from about 15 to about 12 ms, frames with a tick stayed near 17, average 66 to 71 fps. The gain was consistent, about 1 ms per frame in every run. I proposed it as a default.

The owner rejected that, in his words: adding a separate compute pool would fight against the rest of Bevy's pool and feels like a little hack, it maybe works only because the 4 file-loading and 4 background threads were not doing much, a fragile little hack. He is right, and the mechanism confirms it:

- Bevy's compute pool has 16 threads on this box. The sim pool added 12. That is 28 busy threads on 24 hardware threads.
- The gain comes from OS scheduler fairness. Sim threads get time slices that Bevy's workers lose, so frame systems stop queuing behind sim chunks and wait for a core instead, which happens to be shorter.
- It works only while the 8 file and background threads are idle. On an 8-thread laptop the same oversubscription can invert the result.
- Bevy's pool has no priorities, so a second pool was the only lever of that shape. The shape is the problem.

Dropped from the branch on 2026-09-22, backups listed under Commits. If headroom is ever needed, the honest form stays inside the one pool: the sim job spawns fewer, larger tasks so a few workers remain free for systems. Or the sim threads take a lower OS priority, which needs the `libc` crate. Neither is needed now.

### Clean desktop rerun (2026-09-22)

The owner pointed out that this box had the desktop stutter (tmp/handoffs/HANDOFF-desktop-stutter-2026-09-21.md, devlog 0080: a GNOME shell GC freeze every 10 s, fed by MEGAsync) during yesterday's runs. Reran with MEGAsync off, the shell probe clean (0 stalls over 25 ms in 45 s) and the CPU near idle (load 4.6, only a browser). Same view, same recipe, steady-state samples after warm-up.

| | Inline | Pipelined | Pipelined, `FL_SIM_POOL=12` (dropped, for the record) |
|---|---|---|---|
| Fixed tick holds the frame | 15.0 ms | 6.3 ms | 6.3 ms |
| Frame with a tick, p50 / max | 24.8 / 27.9 ms | 16.2 / 18.5 ms | 17.0 / 20.0 ms |
| Frame without a tick, p50 | 9.0 ms | 15.1 ms | 11.6 ms |
| Average frame | 17.5 ms, 57 fps | 15.2 ms, 66 fps | 14.2 ms, 71 fps |

Same picture as yesterday, slightly better in every column, so the stutter did not distort the comparison. The worst regular frame drops from 28 to 18.5 ms. Average frame rate rises 15 percent in this view, and 24 percent with the job on its own threads. The own-pool gain is now clear of the noise: it is the same about 1 ms per frame in all four runs so far. The two cycles of GPU timestamps in devlog 0078 are unaffected by the stutter, they were compared at equal GPU clock.

### Where the main thread frame goes (2026-09-22)

New periodic log line `main thread ms`: three marks (First, end of the fixed loop, Last) plus timers on the two instance data copies. Same view, 200k, medians over one log period. This run carried a little more load than the clean rerun (57 against 66 fps), the split is what matters.

| Leg | Inline | Pipelined | What is in it |
|---|---|---|---|
| Fixed loop, on frames with a tick | 15.0 ms | 6.4 ms | inline: grid 2.8, integrate 6.3, field 0.7, apply and the post-step systems. Pipelined: install swaps (about 0), field 0.8, damage apply, arrows, deaths, fatigue, morale, orders, then the kick's 12 MB of column copies |
| Update and PostUpdate | 6.6 ms | 11.3 ms | instance sync 4.2 against 4.7 ms. Everything else (camera, orders, cards, overlay, arrow sync, audio, Bevy transforms and visibility) 2.4 against 6.6 ms |
| Extract and wait for the render thread | 2.4 ms | 2.4 ms | extract memcpy of the 16 instance buckets 1.9 to 2.0 ms, wait about 0.5 ms |
| Render thread, `write_buffer` | 1.4 ms | 1.5 ms | overlaps the next main frame |

Cross-check: inline frame without a tick = 0.3 + 6.6 + 2.4 = 9.3 ms, measured 9.3. With a tick, 15 more. Pipelined the tick adds 6.4 and the Update leg is 4.7 ms fatter on every frame, which is the even 15 and 16 ms pattern.

The 4 ms swell OUTSIDE the sync was the surprise. Bevy's multi-threaded executor runs systems as tasks on the `ComputeTaskPool`, the same pool the tick job fills with chunk tasks. So the job delays every Update system, not only the parallel sync. That is why `FL_SIM_POOL` helped beyond the sync (plain frames 11.6 against 15.1 ms) before it was dropped. What remains after the own pool is physical core sharing on a 12-core part. Fully resolving it means fewer cores for the sim (lower thread priority, or a smaller pool) or less sim work (items 6 and 7).

The render thread is not the limiter in this view: the main thread waits only about 0.5 ms for it. The GPU works about 4 ms of a 15 ms frame.

For item 2, the per-frame CPU it can remove is now measured: sync 4.5 ms, extract 2.0 ms, write_buffer 1.5 ms, plus the executor swell those systems suffer. About 8 ms per frame at 200k, growing with army size. Devlog 0081 is the detailed anatomy of the frame, with what items 2, 6 and 7 change in it.

## Catch-up clamp

Ported from 1a62029. `Time<Virtual>` max delta is `FL_CATCHUP` tick periods, default 2. Bevy's default 250 ms allows 7 back-to-back ticks after one slow frame, and in pipelined mode every tick after the first in a frame waits for its job in full. Time past the clamp is dropped, so a stall plays as a moment of slow motion. `[catchup] n sim ticks in one frame` logs any burst.

## Commands

```
cargo run --profile opt-dev                      # pipelined (default)
FL_PIPELINE=0 cargo run --profile opt-dev        # inline fallback, bit-identical to before
FL_HASH=60 FL_TEST_DIR=1 cargo run --profile opt-dev
tmp/scripts/hashrun.sh name 40 FL_TEST_DIR=1 && tmp/scripts/hashcmp.sh a b
```

## Final diff review before the merge (2026-09-22)

Read `main...perf/pipelined-tick` whole, 9 files, 785 added, 117 removed. Checked and sound:
- Every system that writes a soldier column runs before the kick, so no write can be overwritten by an install. `apply_reforms` writes only `home`, which the job reads and never swaps back.
- All nine read-modify-write columns swap back, `ammo` included. Positions swap through `pos_prev`.
- The generation guard drops a stale job and recycles its buffers, and a finished job is never lost on an early return.
- The grid configures itself from the positions on every rebuild, so swapping two grid instances is safe.
- `tick_seed` is the same in the job and in the apply, and the same the inline path would compute.
- Terrain never changes in a battle (death craters are disabled), so the `Arc` snapshot clones once.
- The two regiment writes in prep (stand-off anchor snap, `firing`) converge with skirmish's anchor write, no oscillation.
- A panic on the worker surfaces as a crash on the main thread, never a hang.

Found and fixed: the fixed-tick clock runs in every state, so the first pacing samples of a battle carried menu time (9ef6fa6). Cosmetic, diagnostics only.

Noted, not changed: `pos_out` is zero-filled every tick before the integrate overwrites it, about 2.4 MB at 200k. Part of the copy cost item 2 revisits.

## Open

- Owner feel pass DONE 2026-09-21: "a lot smoother at 190k or 200k units fighting each other", "feels alright for gameplay". No complaint about the one-tick order latency.
- A mid-battle restart while a job is in flight is covered by the generation check in code. It was not exercised by a scripted test.
- Not ported from `perf/spikes`: the `[hitch]` whole-tick attribution (800cece) and per-chunk spike attribution (d7596b3). The pacing log covers the need for now.
- Column copies cost about 12 MB per tick at 200k on the main thread. Item 2 (GPU render data) will want the job's buffers as its upload source, which is the moment to revisit what is copied.
