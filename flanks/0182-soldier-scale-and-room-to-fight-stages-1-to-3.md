# 0182: the soldier scale branch, stages 1 to 3 of the room to fight

Written by Claude Fable 5.1.

2026-10-03, Linux, branch `feat/soldier-scale` cut from main 66044d1 in the main tree. Gota approved the start, set the height at 1.70 m, and asked for stages 1 to 3 of `docs/plans/021-room-to-fight.md` back to back before the first play. This devlog holds what was built, how it was checked, and what the measurements say. The short version: the three stages are in and do what their rules say, and the rules alone do not open a fighting mass. The numbers and the reason are below.

## Commits

| Commit | What |
|---|---|
| aa0235b | Every kind 1.70 m tall, with everything that read the old height |
| c2c6bb7 | `FL_GAP=k,m,s,a`, the slot gap per kind |
| 39341f8 | `FL_LOG_ROOM=1`, the room log |
| b34d73c | Rule 1 (the weapon's length) and rule 2 (the clear arc), behind `FL_FIGHT_ROOM` |
| 16f8ec6 | Out-of-formation and walking shares in the room log |
| f0f5f5d | A comment fix |

Strict clippy is clean and the 35 tests pass at the head of the branch.

## Stage 1: the true height

- The height is 1.70 m to the top of the helmet for all four kinds (`half_height` 0.85 in `src/unit_types.rs`). Gota's word was "1.70m"; it was read as the drawn height. If 1.70 m bare-headed was meant, it is the one constant (0.875 for 1.75 m). `FL_UNIT_SCALE` still multiplies it.
- Detail levels: the thresholds are now pixels per metre of soldier height (`LOD_PX_PER_M` 28, 12, 3 in `src/render_units.rs`), so the switch distances stay where they were at the old size. At 1.70 m that is 47.6, 20.4 and 5.1 px. `FL_LOD_PX` still overrides them as soldier pixel heights. The shadow levels follow the same thresholds and came out as before (levels 2, 3, 3 in the log).
- The pose shader (`src/shaders/unit_pose.wgsl`): the rig carries the soldier's half height (`Rig.half`). The lean, the stagger and the fall turn about the real soles instead of 0.5 m under the centre, and the crouch, hop, death sink and wall shield lift are shares of the height. The fallen lie on the ground (`tmp/runs/soldier-scale/s1/fallen2_40s.png`).
- The code-built meshes (`FL_UNIT_MESH=code`) are scaled from the height they were written at (`BUILT_HALF`).
- The selection ring and the move preview's slot circle are 0.5 m in radius (was 0.42): about a man's width, and under half a wall's 1.05 m file pitch so the rings of a wall do not overlap.
- Arrows: the launch point is 0.25 m over the archer's own head, the aim point 0.7 of a man's height, and the strike band, the loft ceiling over friendly blocks and the head clearance over rising ground follow the tallest kind. All were fixed metres tuned for a 1.0 m body.
- The front line marker is drawn 0.4 m over the heads.

Checks:

- Fingerprints against main (a binary built from `git archive main`, `tmp/runs/soldier-scale/bin/flanks-main`): the direction scenario 18 of 18 equal, the wide pile 28 of 28, the two-on-one pile 28 of 28. Archery differs, as it has to: the arrows test a taller body.
- Archery scenario at 36 s: the target unit has 354 of 500 left on main and 217 of 500 at true size, so 146 against 283 killed. Archers kill about 1.9 times as fast until the hit rates are recalibrated (plan 021 puts that after the default changes).
- The mid-air view of the scale try-out (devlog 0176, `tmp/runs/scale/midair/run.sh`, 200k, one 50 s run, not a measurement by the perf rules): 150 fps on the branch. The earlier runs in the same view: 164 at the old size, 99 at 1.64 with the old thresholds, 155 at 1.64 with scaled thresholds.

## FL_GAP

`FL_GAP=k,m,s,a` sets the normal-order slot pitch per kind in metres (knights, men-at-arms, spearmen, bowmen). The default is 1.4 m for all four, so nothing changes without it. Loose order is 1.9 times the kind's gap; the wall stays 1.05 by 1.15 m. A value under 1.4 m is raised to 1.4 m with a warning, because the soft push between men starts there. The army layout of a normal battle sizes every block for the widest kind. The Demo layout still assumes 1.4 m.

Tried with `FL_GAP=1.4,1.4,1.2,2.4`: the warning for 1.2, bowmen at 2.2 m to the nearest comrade, no overlapping blocks (`tmp/runs/soldier-scale/s1/gap_8s.png`).

## The room log

`FL_LOG_ROOM=1` logs every 5 s. For each unit: the distance to the nearest living comrade of the same unit (10th percentile, median, 90th), the median distance to the nearest enemy, the share of men within 3 m of one, the box of the men's centres along the unit's facing and its ground per man, and the shares out of formation and walking. One line per unit while 8 or fewer are alive, and always one line over all units by state (fighting, moving, standing). The box is sensitive to stragglers; the nearest-comrade figures are the ones to compare with the M2TW probe.

## Stages 2 and 3: the two rules

Both sit in `src/sim/soldier.rs` behind `FL_FIGHT_ROOM` (1 by default, 0 for the old behaviour).

Rule 1, the weapon's length. A man closes on an enemy in reach to 0.9 of his kind's reach (knights 1.62 m, men-at-arms 1.8, spearmen 2.16, bowmen 1.44) instead of 1.2 m. A man going to an enemy he sees stops at his own distance between 0.9 and 0.98 of his reach. With the enemy more than 0.3 m inside his distance he steps back to it at the shuffle pace.

Rule 2, the clear arc. Every 8 ticks a man with an enemy in reach looks at the room he has: a comrade in the arc of his cut (a sector on his facing, 1.2 m, 70 degrees to either side), or for a spearman in the lane of his thrust (0.35 m to either side of the line to the enemy, up to the enemy); a comrade within 1.4 m in the quarter to either side of his way to the enemy; any man within 1.4 m in the quarter behind him. With a comrade in the arc he does not start a blow and does not close: he waits, and in some one-second windows (`FL_SIDESTEP`, 0.2) he sidesteps toward an open side.

The `sight` column went from 8 to 16 bits for the two new bits.

Choices made inside the rules, for Gota's yes or no:

- Only a man out of formation steps back. A man still in formation holds his place, and his slot would pull him forward again.
- No step back during a wind-up, and none from an enemy whose unit has broken.
- A wall neither steps back nor sidesteps. Its rear rank is blocked by the man 1.15 m ahead and does not cut.
- "Behind" is blocked by any man, friend or enemy.
- The spear's lane runs along the line to the enemy, not along his facing.

Checks:

- With `FL_FIGHT_ROOM=0` the branch matches main in the three melee scenarios, every fingerprint. The old behaviour is intact in the same binary.
- The spearwall under a charge (`FL_TEST_CHARGE`): the wall keeps its spacing, 1.01 to 1.04 m to the nearest comrade with the rules on and off. At 40 s the wall lane stands at 393 spearmen against 353 knights with the rules on, 373 against 329 off.
- Direction scenario at 36 s: damage per hit by sector is unchanged (21, 27, 49 against 22, 28, 49). Kills by sector front, side, rear: 349, 263, 613 against main's 452, 247, 707.
- Sim cost in the 200k scripted front, one 80 s run each: 10.4 ms per step with the rules on and 9.5 ms off, with 7% more men alive in the first.

## What the measurements say

The two-on-one pile (two 500-man attackers on one 500-man line), the distance to the nearest comrade in the attackers as 10th percentile / median / 90th, and the victims left:

| t | Rules off, attacker 1 | attacker 2 | victims | Rules on, attacker 1 | attacker 2 | victims |
|---|---|---|---|---|---|---|
| 5 s, charging | 0.90 / 1.08 / 1.37 | 0.92 / 1.07 / 1.46 | 500 | the same | the same | 500 |
| 20 s | 0.87 / 1.04 / 1.30 | 0.87 / 1.02 / 1.27 | 357 | 0.84 / 0.88 / 1.21 | 0.85 / 0.89 / 1.29 | 398 |
| 40 s | 0.89 / 1.15 / 1.31 | 0.93 / 1.14 / 1.32 | 207 | 0.87 / 1.10 / 1.31 | 0.87 / 1.07 / 1.34 | 251 |
| 60 s | 0.98 / 1.22 / 1.32 | 1.00 / 1.19 / 1.33 | 80 | 0.89 / 1.16 / 1.32 | 0.88 / 1.13 / 1.35 | 121 |

A 20k AI battle on Big Grassland, the median over the fighting units of their median distance to the nearest comrade: 1.20, 1.23, 1.25, 1.26 m at 30, 60, 90 and 120 s with the rules on, 1.13, 1.25, 1.26, 1.26 m off. Standing units read 1.31 m.

The 200k scripted front, where both armies are under Move orders: 0.79 m with the rules on and 0.83 m off at 75 s, and 3,494 dead against 16,710. Killing is 4.8 times slower there.

M2TW for comparison (devlog 0177): 1.0 m standing, 0.8 m moving, 1.5 m fighting, reached over 30 to 60 s.

Read together:

1. The rules work at the contact. The front rank stands 1.63 to 1.87 m from its nearest enemy (men-at-arms, `FL_DIAG_MELEE`), the mixed band at the contact is thinner (`tmp/runs/soldier-scale/s23/top-room1_40s.png` against `top-room0_40s.png`), and killing is slower: in the two-on-one pile 11, 21 and 51% more victims are left at 20, 40 and 60 s, and in the wide pile 280 against 224 at 55 s.
2. The rules do not open the mass. A settled fight sits at 1.25 m between comrades with the rules on or off, under the 1.31 m of standing ranks and far from M2TW's 1.5 m.
3. The reason is in who the rules reach. They act on a man with an enemy in reach, and only 3 to 28% of an attacking unit are within 3 m of an enemy. The other men are packed by the charge (1.08 m while running, 0.88 m once the front has stopped) and then stand. Nothing they do needs room. Standing men ease apart only until the push between them falls under a planted man's grip, which is at 1.26 m, and that takes about a minute.
4. In a crush under Move orders (0.8 m) nearly every swordsman has a comrade in his arc, so nearly nobody cuts. That is the rule as written, and it is the case the stab comparison in plan 021 was noted for.

So plan 021's expectation, "nearest comrade about 1.5 m, reached in 30 to 60 s", is not met by stages 1 to 3. Nothing beyond the approved rules was built.

## What would open the mass (not built, for Gota)

1. Crowded men make room in steps. A man out of formation who waits behind a comrade keeps the distance he would have stopped at walking up (1.4 m). Crushed closer with open ground behind him, he steps back to it at the shuffle pace, and aside when a side is open. The rear rim opens first and the opening travels forward, which is the direction M2TW's units grow in (21 by 6 m to 28 by 16 m). It uses the distances and the look the sim already has. Feel-critical: it reverses, for men out of formation, the rule that a fighting block never steps back.
2. The stab through the lane. A crowded swordsman whose arc is blocked stabs when the lane to his enemy is clear, as plan 021 recorded. It gives a crush its killing back.
3. The archer recalibration at true size.

## Runs and files

- `tmp/runs/soldier-scale/`: `roomrun.sh` (a muted run with the room log), `room/` (logs and the `[room]` lines of every run above), `s1/` and `s23/` (screenshots), `bin/flanks-main` (main's code, with an `assets` link), `src-main/` and `target-main/` (its build, about 7 GB).
- Fingerprints: `tmp/runs/scripts/gates/ss-main-*`, `ss-s1-*`, `ss-room0-*`, `ss-room1-*`.
- The mid-air run: `tmp/runs/scale/midair/branch_170.log`.
