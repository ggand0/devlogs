Written by Claude Fable 5.1

# 0134: The shipping branch from bee555e: backups, the archive, the crash and the facing rule

Branch `feat/melee-footwork`, HEAD **1dd582f** = bee555e + two commits. Follows 0133 (the measurement that decided this). Gota's decisions: bee555e is the base (the fastest build measured, 148 against 119 on the counter in the locked view, and a behavior he approved in play), the v2 commits are archived for the perf branch, the crowd field's fix is documented but not done here (docs/plans/014-crowd-field-cost.md), and the perf plan lives in docs/plans/013-peak-kernel-pair-pass.md. His bar for this branch: 110 to 120 on the counter in his 200k scenario.

## The moves

- Backups first: branch `backup/melee-footwork-v2-5639239`, ref `refs/backup/melee-footwork-v2-5639239`, bundle `work/backups/melee-footwork-v2-5639239-2026-09-26.bundle` (verified; it requires cc7b08b, which is main).
- `archive/melee-footwork-v2` at 5639239: the touch-box perception, the start events, the mass map, the crowd field and the in-reach melee clock live there, raw material for the perf branch (the touch-box perception is what a pair pass or a GPU scan wants; the crowd field returns through the doc above, built inside the tick job).
- `feat/melee-footwork` reset to bee555e (working tree had no tracked changes; the untracked and ignored files were untouched).
- Two cherry-picks from the v2 commits, rewritten by hand without the FL_FAR_LOOK gate, each verified and committed on its own.

## 8835a90: crash a charge home before the melee begins

From 5639239's regiment logic. A charging regiment that engages keeps its block moving until the enemy has stopped it (the centroid's smoothed forward speed under 0.3 m/s after at least half a second), until its rear has had the time its depth needs at 4 m/s (capped at 8 s), or until it disengages; the charge pace stays on and the melee clock and the contact frame wait. Regiment state in orders.rs (crashing, crash_ticks, crash_cap, adv_speed), the clock and frame conditions and the crash block in frontline.rs, and the pace read in prepare_tick. bee555e's melee clock on wind-ups is kept (no in_reach column).

## 1dd582f: face where he is going once out of formation

From dc41a22's facing rule, one match arm: a committed man with no enemy in reach faces memo_dir (the enemy he remembers, else the fight point), walking or standing. The formed man's rule stays for men in formation. On bee555e a blocked joiner fell through to the formed man's rule and stood facing the line's front.

## Verification (work/scripts/gate.sh on a copy of each build, compared with work/scripts/cmp.sh)

| scenario | crash vs bee555e | crash + facing vs crash |
|---|---|---|
| DIR | 19/19 equal (no regiment charges in it) | 0/19 from tick 60 (committed men's yaw) |
| ARCHERY | 19/19 | 19/19 |
| wide line | 4/29, from tick 300 (the crash) | 4/29, from tick 300 |
| two-on-one | 4/29, from tick 300 | 6/29, from tick 420 |

Statistics (the scenarios are deterministic, so these are exact):

| | bee555e | + crash (8835a90) | + facing (1dd582f) |
|---|---|---|---|
| wide line, victims out of formation at 10 / 20 / 30 / 40 / 50 s | 74 / 357 / 401 / 359 / 303 | 77 / 312 / 346 / 310 / 249 | 77 / 317 / 347 / 306 / 253 |
| wide line, victims alive at 20 / 50 s | 467 / 308 | 416 / 253 | 419 / 256 |
| two-on-one, victim alive at 10 / 20 / 30 / 40 / 50 s | 500 / 413 / 342 / 274 / 213 | 488 / 365 / 271 / 187 / 125 | 488 / 361 / 274 / 202 / 139 |
| DIR at 38 s, kills front / side / rear | 406 / 175 / 653 | 408 / 175 / 653 | 429 / 215 / 635 |
| DIR at 38 s, damage per hit by sector | 20.8 / 27.5 / 49.1 | 20.8 / 27.5 / 49.1 | 21.0 / 27.2 / 49.3 |
| DIR at 38 s, lone rear-charged victim alive | 92 | 91 | 47 |

Reading: the crash brings the whole attacking block in (both pile attackers crash for 148 ticks, stalled by the centroid rule, as on 5639239), so the victims die faster and the wave's share of the living is unchanged (about three quarters of the survivors out of formation at 20 s on both). The facing rule moves DIR the way 5639239 had moved it (426 / 223 / 664, lone 43 there): a man who faces his enemy while waiting opens on him sooner, and the lone victim of a rear charge goes down faster; damage per hit by sector is the same, so the sector model itself is untouched.

The fingerprints and logs of 1dd582f are filed as work/baselines/{dir,arch,pilewide,pile2}-footwork-1dd582f.*, provisional: the refactor's baselines are taken at the commit Gota approves.

## The binary

`target-agent/opt-dev/flanks` is 1dd582f (md5 7bd5968f), clippy clean. Gota's `target/opt-dev/flanks` is still 5639239 and is not touched until he says so; the install is `cp target-agent/opt-dev/flanks target/opt-dev/flanks` with his game closed.

## Open

- Gota's feel check (his step 4): the 5 v 5 first (bee555e's fight point has never been seen there; if it blobs, the crowd field is the fix, inside the job), then the wide line (work/scripts/pile-wide.sh), the two-on-one (work/scripts/pile-two.sh), the 200k in his scenario with the counter.
- The fps on record in his scenario (both armies ordered in) when the box is free; today his own work runs alongside, so no run was made.
- New baselines at the approved commit, then the movement.rs refactor against them (FL_DIAG_REAR and the pile-on debug logging dropped, test scenarios kept), the PR draft, his merge, then the perf branch off main.
- bee555e's tuning stays: the wave, the sidestep chance, the 15 m sight radius.

## Gota's feel check of 1dd582f (the same evening): approved

He ran work/scripts/pile-wide.sh, work/scripts/pile-two.sh, the 5 v 5 and the 200k. In his words: all feel good; soldiers at the far flanks approach the centre enemy in a somewhat staggered way, which feels organic; charging works; in the 200k, 110 to 120 fps at 180k to 200k units remaining, a very occasional dip to 100 when moving the camera around; satisfactory. That meets the bar he set for this branch (110 to 120 in his scenario), so 1dd582f is the approved commit and the fingerprints filed as work/baselines/*-footwork-1dd582f are the baselines the refactor must match bit for bit.

His direction for the refactor: movement.rs is 2,900 lines and unreadable to humans, and the comments carry AI leftovers to trim, "owner" remarks and internal terms such as devlog references. The plan is docs/plans/015-movement-refactor.md.
