Written by Claude Fable 5.1

# 0127: Occupancy bytes and the ring far look (measured, not a win as built)

Branch `feat/melee-footwork`, on top of bee555e (devlog 0124). Gota asked for a perf direction after the sight field (0125) and the collision-grid detour (0126), open to redesigning the neighbor scan or the sight field. This entry records the proposal, the first build of its first two items, the measurements, and why they came out as they did. Nothing is committed; the working tree carries the A1/A2 build described below.

Pictures now live in tmp/notes/vis/ (moved there today; the paths in devlogs 0125/0126, the handoff and the picture scripts were updated).

## The proposal (tmp/notes/perf-direction-2026-09-25.md, tmp/notes/vis/perf-direction.png)

Where the 200k tick's time goes on bee555e, by the previous session's section counters: the separation and reach scan every man runs every tick is ~1250 of 2162 kernel cycles per soldier (the same on main); the footwork added ~380: the 15 m sight scan ~115, the remembered-enemy lookup and join logic ~130, the comrade look ~35, and ~100 in code identical to main's where more men now move. The collision grid rebuild is 4.3 ms wall, 2 to 3 ms of it a serial merge.

The sight field failed because it swept the whole battlefield every tick for a scan worth ~5% of the kernel. The outward ring search over units was dropped because a man with nothing in sight still reads the whole box; the per-cell team bits were dropped because near the fight every box holds an enemy. Each is half of one idea: rings over occupancy bits.

- **A1**: one byte per collision cell (living team 0, living team 1, a team-0/1 man moving over 1 m/s), built with the grid, plus a coarse 6 m level.
- **A2**: the far look reads cells outward from the man's own, checks a cell's byte before its men, stops when the rings left cannot hold anyone closer; ties resolved by grid slot so the answer is the box scan's, bit for bit. The 6 m runner check reads only cells with a moving man of his team.
- **A3** (next, Gota's order): the every-tick separation scan reads the 1.4 m box instead of the 2.0 m box when the bytes of the 2.0 m box hold no living enemy; same pairs in the same order.
- **Tier B**, separate branch off main, not bit-identical: a pair-once separation pass in grid order (estimate -30% of the kernel), and the grid rebuild's serial merge as a parallel prefix.
- Not recommended: the sight field. Backed up on branch `backup/sight-field-on-bee555e` (9eab077, the tmp/patches/sight-field.patch applied on bee555e through a temporary index, working tree untouched).

The wide-line roll-up under the new far look: tmp/notes/vis/wrap-far-look.png (four men of the line, what each reads, the same answer as today).

## What was built (uncommitted, A1 + A2, first version)

spatial.rs: `occ` (one byte per cell, OCC_TEAM0/1, OCC_MOVING0 << team with OCC_MOVING_SPEED2 = 0.98 so the bit never misses a man the runner predicate would accept), built after the scatter pass in a parallel pass over bands of 8192 cells, then `occ_coarse` per 4 x 4 block in a second parallel pass; `nearest_enemy(center, radius, team)`: fine rings 0 to 2 of cells around his own, then rings of coarse blocks, a ring skipped once its gap to his cell squared exceeds the best distance squared by more than a rounding margin, a flagged block expanded to its cells, a flagged cell scanned with the closure's exact distance expression and ties by slot; `any_candidate_vel_in(center, radius, mask, f)` replacing `any_candidate_vel`, skipping cells without a bit of the mask. movement.rs: the far look calls `nearest_enemy` and the runner check passes the moving bit of his team.

Gates (tmp/scripts/gate.sh, compared with tmp/hash-baselines/*-footwork-bee555e.hash): DIR 19/19, ARCHERY 19/19, pile-on wide line 29/29, two-on-one 29/29 fingerprints equal. The behavior is bee555e's to the bit.

## Cost, 200k FL_AUTOSTART, three ABAB pairs, 100 s each, last 10 windows

Both builds carried the kernel cycle counter (tmp/scripts/perf/cyc.py) and the sight-block counter (acqcyc.py). The box was loaded by something else during every run (see the CVAT section): absolute numbers are ~40% above the last session's, the paired comparison stands.

| | bee555e | A1 + A2 |
|---|---|---|
| kernel cycles per soldier | 3007 / 3068 / 3046 | 3000 / 3001 / 3018 |
| sight block, cycles per soldier | 144 / 153 / 150 | 128 / 129 / 130 |
| soldier update wall | 10.57 / 10.95 / 10.68 ms | 10.68 / 10.57 / 10.73 ms |
| grid rebuild wall | 4.17 / 4.32 / 4.32 ms | 4.77 / 4.70 / 4.76 ms |

Kernel about -1%, sight block -20 cycles per soldier, grid +0.45 ms wall. The tick got slightly slower on wall time and about even on CPU.

Per far look (a second pair of builds with counters on the far look, the runner check and the occupancy build; windows 8 to 19 of one run each):

| | bee555e | A1 + A2 |
|---|---|---|
| far looks per tick | 2100 to 5400 | same |
| men read per look | 470 to 645 | 41 to 70 |
| bytes read per look | 0 | 70 to 88 fine + 21 to 24 coarse |
| cycles per look | 4400 to 5900 | 3300 to 4800 |
| runner checks per tick | 0 to 26 | 0 to 46 |
| men read per runner check | 14 to 165 | 0.2 to 48 |
| occupancy build, inside the rebuild | 0 | 0.40 to 0.63 ms per tick |

## Why

Thirteen times fewer men read, 20 to 30% fewer cycles. The model behind the estimate, cost follows the men read, is wrong for this code: the box scan is a streaming pass over contiguous 16-byte records at about 9 cycles per man, two cell-start lookups per row; the ring search hops cell by cell, and every flagged cell costs two lookups into the 1 MB cell-start table (L3 latency), a call and an unpredictable branch, about 100 cycles per hop, so 30 hops cost as much as 300 streamed men. The occupancy build's actual work is ~0.1 ms, but it ran as two extra task-pool round trips on a pool the frame keeps busy; the round trips are the 0.4 to 0.6 ms. The previous session's "cell occupancy skip" cost the same +0.5 ms for the same reason (devlog 0124). The runner check is irrelevant to cost: tens of checks per tick at most.

## The corrected design (proposed, not built)

- A1: build the byte inside the existing scatter pass with one atomic OR per unit (`AtomicU8::fetch_or`), no extra pass, no coarse level. Expected ~0.1 ms.
- A2: the far look walks rows outward from his own row, reads the row's ~21 bytes at once (one or two cache lines), streams each cluster of flagged cells as one contiguous run (two lookups per cluster, then 9 cycles per man), stops at the first row whose gap exceeds the best distance, and narrows the columns the same way. Estimated ~1500 cycles per look instead of ~5000: the sight block from ~150 to ~55 cycles per soldier, 4 to 5% of the kernel, about -0.3 ms wall and -4 ms CPU per tick at 200k.
- A3 keeps its estimate (about -10% of the kernel) with the same caveat: the byte check must be one or two cache lines, and the narrowed scan must stay a streamed run per row. Both estimates carry the risk the first one had until measured.

## The box was loaded: CVAT

The load average was 4.8 before the first run and 10 to 13 between runs. Cause: the CVAT docker compose project (18 containers under /home/gota/work/freelance/cvat, up 11 days) was crash-looping: `cvat_redis_inmem` in "Restarting", `cvat_server` restarting, and every worker container's entrypoint retrying `manage.py migrateredis --check` every few seconds at 2 to 6 cores each. On Gota's instruction (he no longer uses CVAT) all 18 containers were stopped with their restart policy set to `no`, so they stay down after a reboot; containers and volumes were left in place (`docker compose down -v` in that directory removes them for good). The spiral-standalone containers were not touched. Later runs are clean.

## Tools and traps

- tmp/scripts/perf/cyc.py + acqcyc.py apply to bee555e and to this build; the diagnostic counters of this entry are scratchpad-only (diag.py: far look, runner check, occupancy build).
- A comparison build of bee555e was made from `git archive bee555e` with `cargo build --manifest-path <copy>/Cargo.toml --target-dir target`: shares the compiled dependencies. Trap: after it, `cargo build` for the real crate reports Finished in 0.3 s but leaves the OTHER crate's binary at target/opt-dev/flanks (the uplift is skipped because the file is newer). Copy the wanted binary back or touch a source file.
- Trap: a run script that waits with `pgrep -f <name>` on its own binary name matches its own command line and never starts.
- Log the load average before each run; today's runs are the reason.

## Afterwards (same evening)

- Gota: do not commit A1/A2; backed up as tmp/patches/occupancy-bytes-far-look-v1.patch and branch `backup/occupancy-bytes-on-bee555e` (527fb01). The working tree still carries the edits until he says to restore it.
- Gota: no incremental perf; a fundamental fix within this branch's scope, including revising the footwork logic. Two proposals followed, both notes in tmp/notes/ with pictures in tmp/notes/vis/:
  - perf-fundamental-2026-09-25.md (rigid-tiles.png): tiles, rigid-tile reuse of the separation result, sleeping men, dirty-tile rebuild. Rejected for this branch as out of scope (main's systems); kept for the 1M goal.
  - footwork-v2-perception-2026-09-25.md (touch-wave.png): the current plan. A soldier perceives through the touch box he already reads every tick (nearest enemy in it), is told by start events when a comrade of his regiment within 6 m starts running (one small grid query per start in the apply pass, a memory countdown while the roll runs), heads for the fight point beyond that; the 15 m far look and the 6 m runner look go, the memo comes from the scan, lanes fuse into the scan on look ticks. Expected ~1880 to 1900 cycles per soldier at 200k (bee555e 2162, main 1780). The 15 m sight is not M2TW-evidenced (0121: seek radius unknown; 796ff14 chose it against a hollow before any joining mechanism existed). Gota's isolated-man question turned the wave from touch-box-only into start events.
- Handoff for a second opinion by Opus 5.5: tmp/handoffs/HANDOFF-footwork-perf2-2026-09-25.md.
