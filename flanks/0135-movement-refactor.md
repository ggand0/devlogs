Written by Claude Fable 5.1

# 0135: The movement.rs refactor, stage by stage, bit for bit

Branch `feat/melee-footwork`, from the approved 1dd582f (devlog 0134) to **6b5a4e8**, seven commits. Plan: docs/plans/015-movement-refactor.md. Gota's brief: movement.rs had 2,900 lines and was unreadable to humans, and the comments carried AI leftovers to trim, "owner" remarks and devlog references; FL_RECTFIGHT was stale and should go too. Constraint: no behavior change, proven by the four fingerprint gates against work/baselines/*-footwork-1dd582f at every sample; no measurable cost change.

## The commits, each built, clippy clean and gated (DIR 19/19, ARCHERY 19/19, wide line 29/29, two-on-one 29/29 equal)

1. 129e887 Remove the rear-rank diagnostic. FL_DIAG_REAR, its rows, its buffers in the job, its scan inside the kernel and its aggregate: about 350 lines. The FL_LOG_STEP counters it carried moved into SimStats.
2. 2646944 Remove the FL_RECTFIGHT mode. The pass-through pairs with their two grid meta bits and flag columns, the order freeze on engagement with the ordered target, the waypoint around friendly blocks, the anchor freeze and re-form block with its engagement-with-target count and fight origin, the pursuit-range arm that could never fire, the engaged close-ranks period the gate made unreachable, the getter. About 80 sites in six files, 336 lines fewer. The disorder log stays under FL_LOG_DISORDER.
3. 194a1d5 Move the damage apply into its own module: sim/damage.rs with its constants, event types and the DIR sector bookkeeping; the pass reads the same columns in the same order through a small context struct.
4. f73767f Move the tick job and its prep: sim/job.rs, prepare_tick split into copy_columns, archer_fire_solutions, resolve_orders and snapshot_regiments, statements in their original order (the stand-off anchor snap still precedes the read of the anchors).
5. 9462550 Split the soldier's tick into its stages: sim/soldier.rs. One function per stage in the order the tick runs them: begin, drive, scan, look_around, hold_the_frame, acquire, decide_to_join, swing, yield_to_crowd, close_in, steer, face, integrate. Each reads the job's shared inputs through a `Field` (a Copy struct of slices) and the soldier's own row through a `Soldier` (his columns borrowed mutably), with a `Step` carrying what one stage hands the next. The kernel's constants and knobs moved with it. run_tick_job kept the grid rebuild and the chunk loop.
6. ed608f2 Move the tick's plumbing into the sim module: sim/mod.rs (SimPlugin, the resources, the worker and the pipeline, the install and the kick) and sim/diag.rs (the FL_LOG_STEP line, the neighbour audit, the FL_HASH fingerprint, the spike line). Every path elsewhere names `crate::sim` or `crate::sim::damage`; the re-exports went.
7. 6b5a4e8 Say the reason in the comment, not where it was written down: 47 devlog references in 14 files, 9 "owner" remarks, two "Gota's" and 16 mentions of the old file, each rewritten as the rule and its reason; the soldier stages' comments re-wrapped to the line width.

## The shape now

| file | lines | what |
|---|---|---|
| sim/mod.rs | about 500 | the plugin, SimStats, the worker thread and the pipeline, run_tick_job (grid rebuild and chunk loop), step_sim (install, apply, diagnostics), kick_tick |
| sim/soldier.rs | about 1,600 | the thirteen stages with their rule comments, the constants and knobs |
| sim/job.rs | about 480 | TickJob, prepare_tick and its four functions |
| sim/damage.rs | about 350 | the serial apply and the damage model's constants |
| sim/diag.rs | about 150 | the four diagnostics |

The largest function is `swing` at about 260 lines (the state machine with the archers' draw and loose); `scan` is about 160, `close_in` about 155. Everything else is under 120.

## How bit-identity was kept

The bodies were sliced out of the closure by script, not retyped: the chunk-row accesses were renamed to the `Soldier` fields, the shared inputs bound by destructuring `Field`, and the locals that cross stages bound from `Step` at the top of a stage and written back at its end. Rust neither reassociates floats nor fuses multiply-adds, so every expression runs with the same operands in the same order. The two places where the script slipped (a lost dot before a method call, and the archers' `shot` binding shadowing the soldier) were compile errors, not silent changes. The gates ran on a copy of each build's binary while the next commit was being edited.

## Tooling

work/scripts/perf/tickrec.py now patches either layout (the single file up to 1dd582f, the sim/ directory after), and bench4.sh picks the right one, so the cost record of devlog 0133 can be extended on the refactored tree. The cost check of 1dd582f against 6b5a4e8 is in the addendum below.

## Open

- The PR draft (work/drafts/), then Gota's merge.
- The perf branch off main: docs/plans/013-peak-kernel-pair-pass.md, with the fps gate in his scenario.
- Small things seen on the way and left alone: `swing` could split its archer path out; the `yield_to_crowd` and `hold_the_frame` stages take the unused `f` and `s` parameters for uniformity; the pile-test scenario keeps its per-edge log line, which pilesum.sh reads.

## Addendum: the cost check (the same afternoon)

work/scripts/perf/bench4.sh, both builds instrumented with tickrec.py, one 80 s run each in the 200k AI battle, the locked 900 m view, engaged window 30 to 79 s. Gota's own work ran on the box (load 3.8 before the first run, 8.7 before the second), so one run each and the caveats of devlog 0133 apply. Raw logs: tmp/runs/perf4/logs/1dd582f.log and refactor.log.

| build | fps (mean / p50) | grid ms | kernel ms | kernel cycles per soldier | fixed tick p50 | frame with a tick p50 |
|---|---|---|---|---|---|---|
| 1dd582f (before) | 150 / 150 | 4.01 | 6.89 / 6.80 / 7.54 | 1984 / 1985 / 2122 | 6.1 | 9.6 |
| 6b5a4e8 (after) | 156 / 156 | 3.87 | 6.45 / 6.34 / 7.14 | 1841 / 1844 / 1965 | 5.8 | 9.3 |

Not slower. The kernel reads about 7% fewer cycles per soldier after the split, which may be the compiler allocating registers better across the smaller functions or may be one-run noise; either way the `#[inline(always)]` fallback the plan held in reserve is not needed.

Gota's reading after installing the refactored build, his 200k scenario: 125 to 135 on the counter with dips to 110 to 115, against 110 to 120 with dips to 100 on 1dd582f. He notes it may be within noise; it agrees in direction with the cost check above. work/scripts/pile-wide.sh still works on the new layout.

## Addendum: five commit messages rewritten (the same evening)

Five of the earlier footwork commits named Gota in the third person, and two pointed at screenshots under `resources/`, which is not in the repo. Gota asked for them fixed before the PR. The messages were rewritten with `git filter-branch --msg-filter` over main..HEAD after a backup (branch and ref `backup/melee-footwork-pre-msg-rewrite-6b5a4e8`, bundle work/backups/melee-footwork-6b5a4e8-2026-09-26.bundle, verified; filter-branch's own copy is `refs/original/refs/heads/feat/melee-footwork`). Trees are untouched (`git diff` against the backup is empty), so the fingerprint baselines and the binaries stand; only the hashes from the sixth commit on changed. The mapping for the hashes this and the earlier devlogs cite:

| old | new | commit |
|---|---|---|
| 796ff14 | a0b8458 | Seek visible enemies, wait, sidestep and brace instead of following files |
| bee555e | 98df421 | Look at the comrades around every quarter second |
| 8835a90 | c74b730 | Crash a charge home before the melee begins |
| 1dd582f | bc7da87 | Face where he is going once out of formation (the approved commit; the baselines keep the name *-footwork-1dd582f) |
| 129e887 | cb47db7 | Remove the rear-rank diagnostic |
| 2646944 | 448b41c | Remove the FL_RECTFIGHT mode |
| 194a1d5 | e6219ed | Move the damage apply into its own module |
| f73767f | 564bde5 | Move the tick job and its prep into their own module |
| 9462550 | 28a8775 | Split the soldier's tick into its stages |
| ed608f2 | d9a0be4 | Move the tick's plumbing into the sim module |
| 6b5a4e8 | bb45a15 | Say the reason in the comment, not where it was written down (HEAD) |

The archive branch `archive/melee-footwork-v2` keeps the old hashes, since it was not rewritten.

The cleanup commit was amended once more after Gota's last scan: an internal phase name in damage.rs and three history-flavoured comments in soldier.rs and job.rs reworded. Comments only; the binary's md5 still moved (bed16665 to de40139f) because the profile keeps line tables and the comments shifted lines, so the final binary was gated once more: see below. HEAD is bb45a15 (it was 64ba26e before the amend).

Gate of the final binary (md5 de40139f, HEAD bb45a15): DIR 19/19, ARCHERY 19/19, wide line 29/29, two-on-one 29/29 equal to the 1dd582f baselines.

## Merged

PR #9 merged into main as 45a6c87 on 2026-09-26 (branch head bb45a15). Post-merge handoff: work/handoffs/HANDOFF-after-footwork-2026-09-26.md.
