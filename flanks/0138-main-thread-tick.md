Written by Claude Fable 5.1

# 0138: The fixed tick stops looping over the army (perf item 1)

Branch `perf/main-thread-tick` off `main` (fde9a0c). Plan: docs/plans/017-main-thread-tick.md, which also ranks the other perf items and answers the crowd-field question that started this thread (no, it would not be faster at 200k: plan 014 has why).

## Why this item first

At 200k the frame is about 7.7 ms on Gota's counter, and thirty times a second the main thread spends about 5.8 ms inside the fixed tick (devlog 0133's `inside the fixed tick` column). That time lands in the frame at full weight; the sim job on the pool reaches the frame only as contention, under a millisecond. So the fixed tick was the biggest single lever on the counter, and plan 013 already listed its serial per-man parts as required above 200k whatever happens to the kernel.

Where the 5.8 ms went, from the code: the 12 MB of column copies at the kick (1.5 ms, measured in devlog 0081), one serial loop over every man in `update_groups` for the regiment sums (about 1.0), the density field's splat, serial merge and blur (0.8, measured), the death sweep's scan of every man (about 0.5), the ammo tally, the per-man wall flags and the fire solutions' member lists (about 0.5 together), the arrows (about 0.2), and Bevy's schedule.

## The intuition: a handle instead of a photocopy

The sim tick is a job on a worker thread. It needs a snapshot of the soldier columns that the main thread will not change while it computes. Until now that snapshot was a real copy, 17 columns and about 12 MB, copied byte by byte on the main thread at every kick, plus a 2.4 MB output buffer zeroed on top. That is the difference between `Vec::clone`, which copies every element, and cloning an `Arc`, which copies a pointer and bumps a counter: the same memory, read from two places.

The question is then who writes. A snapshot is only a snapshot if nobody changes it under the reader. Copying guaranteed that by construction; sharing guarantees it by timing. The main thread writes the soldier columns in exactly one window, between the install (the job's results become the live state) and the kick (the next job starts), and in that window the job holds nothing. So every write of the fixed tick is a plain write into memory nobody else is reading. The columns are `Arc::make_mut` behind `DerefMut`, so if anything ever does write while a job holds a handle, that write copies its one column first and the job keeps the snapshot it started with; correct either way, and in the fixed tick it never happens.

The job's own results go into its own output buffers, the install swaps them in as the live columns, and the buffers the install releases (the old live columns, unique again once the job drops its handles) become next tick's outputs. Reference counts rise at the kick and fall at the install, every tick, with no cycles anywhere, so nothing accumulates over a long session and nothing is copied or allocated in steady state.

The loops follow the same thought: the job already touches every soldier on the pool, so anything the main thread needs per soldier (the density field, each regiment's index ranges, who died this tick) is computed there and handed over, and the main thread's fixed tick does per-regiment work and swaps.

## What changed

The rule now: the main thread's fixed tick does per-regiment work and swaps, never a loop over the army. What the frame needs per soldier comes from the job, which runs on the pool anyway.

- **Shared columns instead of copies.** The 17 soldier columns the job reads are `units::Column<T>`, an `Arc<Vec<T>>` behind `Deref` and `DerefMut` (`Arc::make_mut`). Every read and write in the codebase compiles unchanged. At the kick the job takes clones; nothing is copied. The main world writes its columns only between the install and the kick, when the job holds nothing, so those writes are plain writes. A write while the job holds a clone (a formation key pressed mid-tick) copies that one column once, which is correct: the job computes from the kick-time snapshot as it always did.
- **Outputs written, not copied.** The read-modify-write columns are read from the clones and written to the job's own buffers; each kernel task copies its chunk's 62 KB in first, then runs the stages as before. The position output is no longer zeroed (every row is written). At the install the job releases its clones, the outputs become the live columns, and the old live columns, now unique, become the next outputs. No allocation, no copy, in steady state.
- **Products of the job.** A `take_tick` system at the head of the fixed tick takes the finished job (waiting for a straggler where the install used to wait) and hands over the density field, built on the worker from the same positions `update_field` read (integer counts, so any chunking is exact; the merge is now parallel over cell strips, per plan 014), and the regiment runs: each regiment's men as index ranges in index order. The job also lists the men whose death countdown just ran out and the living men near their own map edge, widened by the charge knockback the apply pass can still add.
- **The loops, on the runs.** `update_groups` sums per regiment in parallel, each task walking its regiment's runs in index order, the same additions in the same order as the old scan of the army. The ammo tally walks the archer regiments. The fire solutions' member lists are built on the worker. The death sweep visits only the job's candidates, reproducing the old swap-remove sequence exactly (a hole takes the last man and is examined again in place). The per-man wall flag is gone: the grid rebuild reads the regiment's flag through the regiment index it already has.
- `FL_PIPELINE=0` and the first tick of a battle go through the same prep, run and install; with no job to take, the runs are built inline and `update_field` rebuilds the field inline.

Files: src/units.rs (the column type), src/sim/job.rs, src/sim/mod.rs, src/frontline.rs, src/spatial.rs, src/combat.rs, src/arrows.rs, src/sim/damage.rs (the knockback margin).

## Verification

Builds and clippy clean in `target-agent` (the project binary was not touched). The baseline is `main` at fde9a0c built the same way into the same target dir, kept as tmp/runs/perf5/bin/flanks-main next to the branch binary flanks-item1.

**Fingerprints (FL_HASH=60, work/scripts/gate.sh):** every fingerprint equal, baseline against branch, in DIR (19), ARCHERY (19), the wide pile (29) and the two-on-one pile (29). The inline path too: `FL_PIPELINE=0` on both binaries, DIR and ARCHERY, 19 of 19 each. The first inline comparison was made against the baseline's pipelined hashes and read 0 of 19, which is the known one-tick difference in the regiment snapshot between the two modes, not a defect; like against like is exact.

Committed as 95e1963, with the release-at-the-take fix as a second commit after the review of the diff against main.

**Cost (200k AI battle, camera locked at 900 m, culling off, muted, 80 s each, the bench4 recipe; window from 30 s after the first fps line; work/scripts/perf/pace.py over the periodic log lines).** The pair ran back to back in a quiet window with a watcher for other games: none appeared. Two earlier attempts were refused by the launcher because the graphics tree's game was running at that moment, which is the wrapper doing its job; the logs of those attempts were overwritten by the pair below.

| run | fps (mean / p50 / p10) | frame ms | inside the fixed tick p50 | frame with a tick p50 | frame without a tick p50 | grid | kernel | field | update+post p50 |
|---|---|---|---|---|---|---|---|---|---|
| main (fde9a0c) | 150 / 150 / 144 | 6.68 | 5.8 | 9.4 | 4.5 | 3.83 | 6.61 | 0.76 | 2.7 |
| branch (95e1963) | 186 / 187 / 177 | 5.39 | 1.7 | 5.3 | 3.9 | 3.60 | 6.69 | 0.65 | 2.6 |

The fixed tick on the main thread fell from 5.8 to 1.7 ms, about the 1.5 ms the plan estimated, and the counter rose from 150 to 186 in this view. The kernel is unchanged (it is the same code); the field's 0.65 ms is now worker time inside the job, no longer on the main thread. Frames without a tick also shortened a little, which is the pool being free of the main thread's field splat and the snapshot copies. Logs: tmp/runs/perf5/logs/, binaries tmp/runs/perf5/bin/.

What is left in the 1.7 ms: the install swaps, the damage apply, the arrows, the sweep over the candidates, fatigue, morale, order clearing, the regiment snapshot and fire solutions of the kick, the parallel regiment sums (a scope over 200 tasks), and Bevy's schedule for the fourteen systems of the fixed tick. A per-system timer line would split it further; not taken here.

## Open

- Gota's counter in his own all-in scenario is the number that decides (the record above is the lenient AI battle in the locked view).
- `fill_vacated_slots` still loops over the army on ticks with losses in formed regiments (it runs after the sweep, on the changed layout). Small; on the list if a per-system timer shows it.
- The alive count is still one pass over two byte columns.
- Next on the perf branch per plan 017: the band grid rebuild, then the pair pass (plan 013).
