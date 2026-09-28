# 0158: A retargeted regiment's men answer the men who come at them

Written by Claude Fable 5.1. Date: 2026-09-27. Branch fix/melee-orders, on top of bc98607 (handoff work/handoffs/HANDOFF-melee-orders-2026-09-27.md, devlog 0153 for the design history).

## The problem

Gota played 5efb0e4's retarget melee: a regiment ordered onto a new target mid-fight got there, but its men then ignored the old enemy's men "even if one is right behind him until the very last moment", and the rule read as robotic. He wants the order followed without half the regiment staying with the old enemy, and without men blind to a man at their back.

## Why 5efb0e4 read that way

Its `focus` filtered what a man perceives by regiment identity everywhere beyond weapon reach: the far look, the remembered enemy and the closing step all refused old-enemy men. The scan box is 2 m and reach is 1.8 to 2.4 m, so "self-defence" only began at sword length, in practice when a chaser's blow had already landed in the back. The second tell was the join walk: a man kept walking to the fight point at a quarter pace while winding up at someone else.

## The rule now

A man of a regiment fighting its new target strikes whoever is in reach in front of him, sticky to the man he fights, as in any melee. On his far look he notices two kinds of men: the target's men anywhere within his 15 m sight, and any enemy within his threat distance (4 m, FL_THREAT_R) who is coming at him, closing faster than 1 m/s (GOING_SPEED, the constant a comrade "running to the fight" already uses). He remembers a man he noticed or fought while that man is within 4 m, then lets him go. Standing proximity never pulls him: an old-enemy line 3 m off his flank that does not come at him is not his business. Everything else, closing, facing, waiting, sidestepping, joining, is the footwork unchanged.

The regiment layer is 5efb0e4's: the Move-like break-off, only target strikers starting the new melee and laying its frame, the fight point on the target while it has men.

## The code

sim/soldier.rs, against main the focus touches four places: the far look (`acquire`: a focused man walks the same 15 m cells with the velocity-reading candidate walk and accepts a target man, or a man within 4 m coming at him; an unfocused man runs the old closure, in its own branch), the remembered enemy (`look_around`: a non-target memo is valid within 4 m), the closing step (`close_in`: the same rule, on the distance the branch computes anyway) and the `moving` gate from 13217d2. The scan's target preference, its sticky exception and the wind-up join walk of 5efb0e4 are gone: the scan and the join gate are main's again.

frontline.rs: FL_RETARGET_FOCUS=0 is the A/B switch. Off, the break-off ends the tick the new melee starts and the men fight whoever they meet, the behaviour of dadd8c9 that split a regiment in half.

## The scenario

FL_TEST_RETARGET=1 (Scenario::Retarget, regiments.rs `spawn_retarget_test`): blue R (500 light) attacks orange A head-on, orange B stands under no order 60 m to the side of A (FL_RT_B_DIST; the blocks are 34 files wide, so the near flanks are 12 m apart). At 20 s (FL_RETARGET_AT) the hook in orders.rs orders R onto B and logs every 5 s how many of R's men strike A men, strike B men, stand within 2.5 m of an A man, and R's distance to both. A attacks R at spawn so it follows R: the pursuit case. FL_RT_A_HOLD=1 keeps A in hold: the standing case. The hook fires on the sim tick, so two runs of one binary retarget on the same tick; FL_RETARGET_AT alone runs the same hook in any battle (every engaged player regiment onto the nearest other enemy), for the 200k worst case.

## Cost, by reading

Unfocused regiments (every AI regiment, every player regiment not retargeted): the scan has fewer instructions than 5efb0e4, everything else is the same instruction path behind a short-circuit on focus. A focused man's far look reads each candidate's velocity and runs one distance and one closing test more, about 5 cycles on a read that costs about 9 (devlog 0128). Worst case, every man of a 200k army in a focused regiment and looking at the full rate: 2.6M far-look reads per tick times 5 cycles = 13M cycles, over 12 pool threads at 4 GHz about 0.27 ms on the job per tick, 4 percent of the 7 ms kernel, and the tick is pipelined so that reaches the frame at about a twelfth. A retargeted regiment's men fight instead of walking away, which is the cost they had before the retarget.

## Measurements

(filled in below from tmp/runs/retarget/logs)

Determinism: two runs of the new binary in the Retarget scenario (FL_HASH=60) retarget on tick 600 both times and give 19/19 equal fingerprints and identical count lines (tmp/runs/retarget/logs/det1, det2).

Fingerprint gate, new binary against the bc98607 build (work/scripts/gate.sh, no retarget in any of the four): dir 19/19, arch 19/19, pilewide 29/29, pile2 29/29 equal. Every regiment without a retarget runs the same sim bit for bit.

### Behaviour A/B in the Retarget scenario

500-man light regiments, R retargeted at 20 s. old = bc98607 with only the scenario compiled in, so 5efb0e4's sim; new = this build; nofocus = this build with FL_RETARGET_FOCUS=0, dadd8c9's rule. Cells: R alive, men striking A / striking B, men within 2.5 m of an A man, R's distance to B in m. Logs tmp/runs/retarget/logs/beh-*.log, table by work/scripts/perf/retarget-summary.py --compare.

| run | 30 s | 45 s | 60 s | R alive at 70 s | A alive 70 s | B alive 70 s |
|---|---|---|---|---|---|---|
| new, A pursues | 240, 6/4, 52, 37 | 124, 6/4, 25, 31 | 56, 0/3, 0, 24 | 13 | 181 | 397 |
| old, A pursues | 240, 12/9, 53, 35 | 137, 0/7, 13, 25 | R broke at 55 s with 65 men | | 217 at 55 s | 427 at 55 s |
| nofocus, A pursues | 237, 3/8, 55, 39 | 132, 4/4, 36, 39 | 52, 3/0, 12, 44 | 32 | 148 | 422 |
| new, A holds | 299, 9/2, 58, 42 | 214, 4/1, 38, 35 | 146, 2/4, 9, 29 | 120 | 165 | 405 |
| old, A holds | 293, 6/8, 40, 38 | 230, 2/10, 19, 32 | 186, 0/4, 0, 29 | 168 | 222 | 402 |
| nofocus, A holds | 289, 10/4, 64, 48 | 195, 11/4, 56, 48 | 120, 7/1, 37, 49 | 86 | 101 | 437 |

- Pursuit: every variant loses R at the two-on-one rate of the pile2 gate scenario (about 1.35 enemy men per own man lost). new and old both move R onto B (43 to 24 m); nofocus lets A drag it back (39 to 44 m, 12 men still at A's side at 60 s). new's men answer the chasers and kill more of them (A 181 at 70 s against old's 217 at 55 s); old's R broke at 55 s with 65 men while new's fought on, its melee exchange holding morale.
- Hold: new and old both peel R off A completely and fight B at 28 to 29 m; nofocus never leaves A (37 men at its side at 60 s, 48 m from B). old peels faster (nobody at A by 60 s against 75 s) and cheaper (R 168 against 120 at 70 s): its men walk off mid-swing and A, on hold, does not follow. new's men at sword length stay in their duels until one side dies, about a fifth of the regiment for the first 30 s, and kill 57 more A men for 48 more of their own. That lingering front is the identity-blind reach rule; a withdrawal step out of reach (M2TW's ACT_WITHDRAW) is the lever if it reads wrong.

### 200k AI battle, every engaged player regiment retargeted at 45 s

FL_AUTOSTART FL_DEPLOY=0 with FL_RETARGET_AT=45, camera locked at 900 m, logs tmp/runs/retarget/logs/perf-*.log. 17 blue regiments of about 750 men were ordered onto the nearest other orange regiment, 16 to 17 m away in the contiguous orange line. None of them reached its new melee in the 65 s left (new build 0 of 17, old build 3 of 17): their slots marched into B's block (centroid distance 17 to 7 m) but the blocks themselves barely moved, jammed in A's mass with about 200 men within 2.5 m of an A man throughout, and the men striking B never reached the lock threshold (3 percent of the regiment, 22 men winding up at B men on the same tick), so the melee clock stayed at 0 and the regiments stayed in the Move-like break-off: formed, no fight point, no join wave, striking only the A men in front of them, cut down from 750 to 300 in 60 s while the old target went 823 to 417 and B 986 to 874. Team totals at 96 s: blue 90546 / orange 91756 against 92620 / 93784 in the plain run.

This is 5efb0e4's regiment layer, identical on both binaries, and it makes these runs a measurement of the break-off's cost, not the focus melee's: sim step 6.45 ms (new) and 6.82 ms (old) against 6.93 and 6.84 ms plain, all inside the AI battle's run-to-run spread. The cost case needed a scenario where the focus melee starts, hence the 48-triplet run below.

Proposed, not done (feel-critical, Gota's call): a retargeted regiment whose block the enemy has stopped, the crash phase's own condition (frontline.rs `v_fwd` under the step pace for a second), while its men fight anyone at the lock threshold, ends its break-off where it stands and starts the focus melee there: fight point on B, the join wave, and the focus rule taking the men to B's men and answering A's. Today it stays a formed block being chewed.

### The cost case: 72 triplets of 500, 108k men, every R retargeted at 20 s

FL_TEST_RETARGET=1 FL_RT_N=72 FL_RT_SIZE=500, camera locked at 900 m, FL_LOG_STEP=1, logs tmp/runs/retarget/logs/mid-*.log. All 72 retargeted regiments (36k men) enter their new melee within seconds here, unlike the 1000-man layouts: the lock threshold is 9 to 15 men, and the flanks of A and B are 12 m apart. The blue army breaks after 55 to 65 s, so the window is 30 s to the end, and the fight kills about 1000 men a second, so the step time is also given per 100k live units. old = bc98607's sim with the scenario compiled in (its focus: the scan preference, the reach-only self-defence, the wind-up walk); nofocus = this build with FL_RETARGET_FOCUS=0, a plain melee after the break-off.

| run | sim step ms mean / p50 | step ms per 100k live units mean / p50 | [step] 5 s means from 30 s | fps mean |
|---|---|---|---|---|
| new | 3.18 / 3.22 | 5.62 / 5.35 | 3.73 3.56 3.30 3.18 2.86 2.49 | 227 |
| old | 3.58 / 3.56 | 5.98 / 5.82 | 3.93 3.81 3.61 3.34 3.00 | 230 |
| nofocus | 3.18 / 2.98 | 5.76 / 5.53 | 3.75 3.48 3.20 3.11 2.86 2.41 2.50 | 233 |

At every matched 5 s window the new rule's kernel is at or under the plain melee's and under 5efb0e4's. The bound Gota approved the plan on (at most 0.27 ms more on the job per tick at 200k, every man looking) is not approached: the measured difference against the plain melee is within noise and on the cheaper side. The 144k 1000-man run (48 triplets, only 6 regiments in a focus melee, the rest jammed in the break-off) reads the same way: step per 100k live units 4.50 (new), 4.64 (old), 4.70 (nofocus).

## State

Commits 255a750 (the rule and FL_RETARGET_FOCUS) and 2a5e612 (the scenario and the tick-based hook) on fix/melee-orders, unpushed. Gota has not played it. Handoff: work/handoffs/HANDOFF-melee-orders-2026-09-28.md.

## Play check and discussion, 2026-09-28

Gota played the 20k battle with both toggle values, retargeting regiments over and over. It feels mostly the same, which is expected: the toggle changes nothing in the break-off (the dashed-line march is identical), only what happens once the new melee starts, and only where old-enemy men stand within 15 m of men with nobody in reach. In an open battle that is the exception. The two differences he saw are exactly the targeted cases: with the rule on, more men reach B when the regiment is between two close enemies (resources/combat/debug1.png: regiment 18 as a 9-file column threading into the gap between two orange regiments, still in the break-off); with it off, dadd8c9's split came back, the men engaging A never engaging B. He keeps the new rule.

Two clarifications from the discussion. First, the 5.76 in the cost table is this binary with the toggle off, a plain melee after the break-off, not main; main never ran, it has no break-off, and its only equivalent measurement is the gate showing every regiment without a retarget unchanged bit for bit. Second, 5.62 against 5.76 is noise, the 5 s kernel means sit on top of each other; the claim is "no measurable cost over a plain melee", not "cheaper". Confidence rests on three legs: the added work is one block that runs for a focused man with nobody in reach once per 8 ticks (the same cells plus the velocity stream and a few multiplies per candidate), everyone else runs the old instructions behind a short-circuit, and three sizes measured no difference beyond noise. Gota's summary of the change is the right one: 5efb0e4's regiment layer and its three identity gates, each widened by "or any man within 4 m coming at me", minus the scan rule that a target man in reach beats the man you are fighting and minus the quarter-pace walk while winding up at someone else.

His own 200k reading from the flank lookout: 115 fps with the lines blobbed up, 105 to 108 after retargeting nearly every regiment across the field, the same on main and on this branch. So the drop is what a mass retarget does to the battle, not the rule. Two candidate mechanisms: render, marching blocks walking into the near shadow cascades and the frustum of a view that was static; and sim, thousands of men of fighting regiments with nobody in reach each running the 15 m far look every 8 ticks, a class devlog 0128 measured at up to 1.5M reads per tick, though the job's weight on the frame is about a twelfth. The 2 s log line separates them without a build: drawn count and the gpu pass lines against the step ms, before and after the retarget.

To keep in mind: this behaviour has been checked in the scenario, at 20k and once at 200k. It needs watching over the long term, in more situations (adjacent lines, pursuit by several regiments, cavalry when it comes, hold mode), and Gota may change his mind on the feel; the toggle stays in for that. The open items stand: the lingering duels at the old front, and the break-off that never ends in a packed line.
