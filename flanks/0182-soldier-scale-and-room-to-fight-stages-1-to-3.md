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

## 2026-10-04: the first play, two wrong readings, and what Gota saw

Gota played the branch overnight. First impression: better than the 1.64 try-out before the branch, and still pretty tight. Screenshots, named by Gota: `refs/unit_scale_branch/` (`looks_tight.png`, `looks_fine0.png` to `looks_fine2.png`, and two annotated ones, `debug0.png` and `debug1.png`, with "packed?" and "fine?" areas circled).

### The hypotheses, in order, and what became of them

1. Plan 021's hypothesis: the two rules open a fight to about 1.5 m between comrades. Wrong, measured above: a settled fight stays at 1.25 m with the rules on or off.
2. My reading of `debug1.png`: the strip where the lines fight is packed because the footwork lets a second-rank man step into the 1.4 m gap between two front men, where he ends 0.9 m from each. Wrong, and stated without checking the geometry. A man walking to an enemy waits when a comrade is within 1.4 m of him and within 45 degrees of his way. In the middle of a 1.4 m gap both front men meet that test from about 1.2 m behind their line, so he stops there. A gap has to be about 2 m wide before he walks through it. The sketch `work/notes/vis/031-contact-band.png` was drawn from this wrong reading; it has been redrawn to show only what is measured.
3. The proposal that followed from 2, to give the footwork the arc's measure (1.2 m): withdrawn. The footwork does not cause the packing.
4. Still open: how much of the strip's 0.85 to 0.9 m comes from the charge and the approach (the formation drive pushing every man forward until the unit holds at contact) and how much from closing on an enemy already in reach, which ignores comrades. Not measured.

### Gota's observation

A 20k battle on Big Grassland with no rear units shows no problem. In 200k, soldiers of rear units on their way to an enemy unit go through the unit in front of them, and the men behind push the men ahead as they go. That is where it gets tight. The pictures agree: the "packed?" areas in `looks_tight.png` and `debug0.png` are regular rows several units deep with no gap between units, men still in formation, behind a unit that fights.

### What the code does (read 2026-10-04)

- An attack order lays the unit's slots around the target's centre (`Groups::goal`, `resolve_orders` in `src/sim/job.rs`). Every man walks straight to his slot (`drive`). With a friendly unit fighting that target, the slots lie beyond it.
- The wait behind a comrade exists only for a man out of formation going to an enemy (`close_in`). A man in formation walks into whoever is in his way; only crowding fades his drive (`yield_to_crowd`).
- The unit stops chasing only when it holds a contact frame, which needs about 3% of its own men fighting (`frontline.rs`). A unit behind a friendly unit seldom gets there and keeps pushing.
- The enemy AI sends every idle unit at the closest player unit with a penalty per unit already on it, and no look at what stands between (`ai.rs`). Any idle unit not on Hold attacks an enemy unit within 40 m.
- The pile test's own acceptance text asks for this: "the second wave stands pressed against the fighting mass (not parked at parade pitch ...)" (`spawn_pile_test`). It was written at the old soldier size.

### The rule Gota approved to try (2026-10-04)

Gota: "let's try that first, though I'm not sure yet about what risks would it bring".

A man in formation walking to his slot under an attack order waits behind a comrade who stands in his way; he does not walk into him. Details as built are in the next section, with the measurements.

Risks named to Gota before the start: a unit ordered to attack through a fighting friendly unit queues behind it where today it pushes through; the crash of a charge gets softer because rear ranks stop shoving; Move orders stay as they are; the sim cost rises because every marching man of an attacking unit takes the quarter-second look.

### The arc overlay

`FL_DEBUG_ARC=1` (commit 1e9e261, `src/arc_overlay.rs`) draws the arc of the cut under every sword soldier within 30 m of the camera's focus: a grey outline with no enemy in reach, green with an enemy in reach and a clear arc, red with a comrade in the arc. Spearmen show the lane to the enemy in reach. `FL_DEBUG_ARC=<metres>` sets the distance. About 120 fps against 200 in the two-on-one pile. The direction and wide-pile fingerprints match the build before it. Sample: `tmp/runs/soldier-scale/arc/pile2_22s.png`. Figure of the two tests: `work/notes/vis/029-arc-and-spear-lane.png`.

### The rule as built, and what it measured (617f057, 2026-10-04)

`wait_behind` in `src/sim/soldier.rs`, on with `FL_FIGHT_ROOM`, off on its own with `FL_WAIT_BEHIND=0`. In the quarter-second look, a man in formation under an attack order (not a Move order, not out of formation) checks for a living comrade within 45 degrees of his way to his slot who is not walking the same way (under 0.5 m/s along it):

- a man of another unit holds him up from 1.4 m, when that unit is fighting or the man is himself held up (his sight bit at the tick's start, the new `Field::sight_prev`);
- a man of an idle unit does not hold him up, so an attack still passes through a unit that stands idle;
- a man of his own unit holds him up from 1.1 m, so a unit still steps off together;
- once held up he stays so until that man is 1.4 m away or walks on. Without this the rear ranks eased apart to 1.26 m, were released at 1.1 m, ran in again and packed again.

A held man stands (`hold_the_frame`, the existing "comrade between him and his mark" bit).

Checks: `FL_FIGHT_ROOM=0` matches main in the direction and two-on-one scenarios; `FL_WAIT_BEHIND=0` matches the build before the rule in the direction and wide-pile scenarios. Strict clippy clean, 35 tests pass. Sim step at 200k: 9.5 ms on, 9.6 ms off, no difference.

Six attackers in two rows on one unit (`FL_TEST_PILE=1 FL_PILE_N=6`), median distance to the nearest comrade:

| | Front row (3 units) | Rear row, sides | Rear row, centre |
|---|---|---|---|
| Rule off, 20 to 60 s | 0.85 to 0.87 m | 0.84 to 0.86 m | 0.82 to 0.85 m |
| Rule on, 20 s | 0.89 to 0.90 m | 0.98 to 1.00 m | 0.87 m |
| Rule on, 60 s | 0.95 to 0.97 m | 1.06 to 1.09 m | 0.88 m |

The 200k AI battle on Grassland, one 150 s run each (the AI battle is not deterministic), the median over fighting units, with the lowest unit:

| t | Rule off | Rule on |
|---|---|---|
| 30 s | 0.87 m (0.80) | 0.91 m (0.87) |
| 60 s | 0.85 m (0.79) | 0.92 m (0.86) |
| 90 s | 0.81 m (0.79) | 0.88 m (0.86) |
| 120 s | 0.84 m (0.79) | 0.98 m (0.87) |
| 145 s | 0.86 m (0.79) | 1.04 m (0.87) |

Units that wait behind a fight read 1.16 to 1.25 m after 90 s with the rule on; with it off the units still moving read 0.79 to 0.88 m.

Read: the rule stops the pushing. The tightest unit goes from 0.79 m (bodies pressed past the 0.9 m body distance) to 0.87 m, waiting units ease apart, and the fighting units gain 0.04 to 0.18 m. It does not solve the packing: fighting units stay near 0.9 to 1.0 m.

Why it is not enough. The packing is made before anyone stands still. Every unit attacking one target lays its slots around that target's centre, so the slot grids of several units lie on the same ground. On the approach the men are all moving the same way, which the rule lets pass, and the units merge side by side and one into another. The crowd's own brake stops them at about 0.87 m: the drive fades between a crowd figure of 1.2 and 2.5, which is six neighbours at 1.12 m and at 0.82 m, values set when a body was 0.6 m wide. Once stopped, a rank eases apart front to back but not sideways, because the pushes from the two sides cancel; only the flanks of the whole mass can open. The six-attacker pile shows it: the centre rear unit, squeezed between the two side units, stays at 0.87 m for the whole run.

So the cause that remains is at the unit level, in where an attack order puts a unit: on the target's centre, however many friendly units are already there. Not built, for Gota:

1. A unit's attack frame takes free ground. It does not lay its slots where a friendly unit's slots already are: it comes up beside or behind them.
2. The crowd's brake at true size. The two crowd figures scaled to the true body, so men stop pushing at shoulder width, about 1.1 m, where today they stop at 0.87 m.
3. The AI holds its rear units back until a unit in front needs relief.

## 2026-10-04, later: the wait rule rejected, and Gota's idea of pushing the line

- Gota played the wait rule (617f057) and stopped after 20 seconds: "all the soldiers in rear look frozen and lifeless". Going around a unit or forcing through it are both better than waiting. The rule was reported with numbers only; no picture of the rear units was looked at before the report. Removed in 23722f1; the direction and wide-pile fingerprints match the build before the rule.
- Asked for the best solution, Claude proposed unit paths around friendly units on the planned pathfinding. Gota rejected it: it relies on something that does not exist on this branch and whose cost is unknown.
- Claude then proposed a crowd brake at the true body (men stop shoving at 1.15 m). Gota's answer: that is another spacing limit like the 1.2 m clearance, and a press may be packed at 0.9 m for a while. The real fault is that the push of the rear soldiers does not travel on to the enemy's side and the defender's rear ranks; a heavier attacker should push the defender back, as at Cannae and as Gota saw in M2TW defending a breached gate. The brake was withdrawn.
- The idea is written up as `docs/plans/023-pushing-the-line.md`: pressure travels through the press, the frames follow the line, and mass, depth and a braced wall resist. It lists the historical cases (from memory), the code that stops the push today, the risks and what Gota has to decide. Not approved to build.
- The branch after 23722f1: the true height, `FL_GAP`, the room log, the two fight rules behind `FL_FIGHT_ROOM`, and the arc overlay `FL_DEBUG_ARC`.
- Why the press went unnoticed in 0.2.1: the sim is the same (with `FL_FIGHT_ROOM=0` the melee fingerprints equal main's, and nothing in the sim changed between 0.2.1 and main), so the men stood 0.8 to 0.9 m apart behind a fight then too. A man-at-arms was drawn 0.59 m wide at his widest (arms, shield and weapon), which left 0.26 m of air at 0.85 m; at 1.70 m tall he is 1.01 m wide there and the same 0.85 m is 0.16 m of overlap of arms and shields. The torsos (0.44 m) stay 0.41 m apart, which is what Gota saw in `refs/unit_scale_branch/debug2.png`; the first version of the figure drew the whole 1.01 m as body and was corrected. Figure, with the same moment of the six-on-one pile drawn at both sizes (`FL_UNIT_SCALE=0.588` against the default, same camera): `work/notes/vis/032-same-press-two-sizes.png`. The widths quoted earlier in this devlog (1.07 to 1.15 m) are the models at their built 1.80 m; at 1.70 m they are 1.09 m (knight), 1.01 m (man-at-arms), 0.94 m (spearman) and 0.75 m (bowman).
- Gota, after the figure: the simple fix is either a wider body distance than 0.9 m or the limit at 1.15 m; plan 023 is probably a topic for another branch.

## 2026-10-04, evening: the body distance at 1.0 m and the shove limit

Gota's order: try the body distance at 1.0 m first, then the shove limit, as separate commits. Gota's reading of the limit: a man who wants to get somewhere goes through the crowd, and keeps some distance from the men who fight instead of going in shoulder to shoulder.

For the record: the 0.9 m body distance, the 1.4 m separation distance and the crowd's two brake figures (1.2 and 2.5) all date from the first movement commit, b986e15 of 2026-07-08, when a soldier was a cube 0.62 m wide, and had not changed since. M2TW's value is 0.8 m (a collision radius of 0.4 m, devlog 0177). At 1.70 m our soldiers are 1.09 m (knight), 1.01 m (man-at-arms), 0.94 m (spearman) and 0.75 m (bowman) across with arms, shield and weapon; the torso is 0.42 to 0.54 m.

- a8dbdf6, the body distance: 1.0 m (`body_distance` in `src/sim/soldier.rs`), the most a wall's 1.05 m files allow. `FL_BODY=<metres>` sets it; `FL_FIGHT_ROOM=0` keeps 0.9 m.
- 3ab4fc5, the shove limit (`shove_limit`): a man's own drive fades as the men around him close in and is gone when six of them stand 1.2 m from him, halfway between the separation distance and the body distance; the fade starts at 1.3 m. In a square rank with four neighbours the same figure is reached at 1.1 m. The crowd's two older figures still weaken the push between bodies and damp a jam, as before. `FL_SHOVE=<metres>` sets the limit, `FL_SHOVE=0` turns it off.

Checks: `FL_FIGHT_ROOM=0` matches main (direction, two-on-one); `FL_BODY=0.9` and `FL_SHOVE=0 FL_BODY=0.9` match the build before each change (direction, wide pile). Strict clippy clean, 35 tests pass.

Distance to the nearest comrade, median:

| | Before | Body 1.0 m | Body 1.0 m and the limit |
|---|---|---|---|
| Six-on-one pile, attackers, 20 to 55 s | 0.81 to 0.87 m | 0.90 to 0.95 m | 0.99 to 1.10 m |
| The tightest tenth there | 0.77 to 0.83 m | 0.82 to 0.91 m | 0.96 to 0.99 m |
| 200k AI battle, fighting units, 30 to 100 s | 0.82 to 0.87 m | not run | 1.00 to 1.08 m |
| 200k, the tightest unit | 0.79 to 0.83 m | not run | 0.98 m |
| 200k, units on the move | 0.80 to 0.94 m | not run | 0.98 m |

Pictures, looked at before the report: `tmp/runs/soldier-scale/body/pile6-body0.9_30s.png` against `pile6-shove_30s.png` (the same pile, the same camera), and `ai200k-base_105s.png` against `ai200k-new_105s.png` (the 200k battle from the field's centre; the AI battle is not deterministic, so not the same moment). Before, the mass is a carpet of bodies with no ground showing; after, there is grass between the men everywhere and single men can be told apart. It is still a dense crowd.

What it did not do: the mass rests at the body distance, about 1.0 to 1.1 m, not at the 1.2 m limit. Men are still carried in by their own speed and by the units converging on one target, and once stopped a rank does not open sideways.

Side effects measured:

- The charge presses less. In the spearwall charge test the charged line's centre moved back 0.8 m by 40 s (1.3 m before) in the wall lane and not at all in the open lane (1.3 m before). The wall keeps its spacing (1.04 to 1.05 m).
- Killing is a little slower again: 122 victims left at 55 s in the six-on-one pile against 71 before.
- The direction test is about the same: 372, 201 and 628 kills by front, side and rear at 36 s (349, 263, 613 before).
- Sim step at 200k: 8.2 to 8.4 ms against 8.9 to 9.1 ms.

### Gota's play of the shove limit (2026-10-04, night)

- The soldiers moving up to close a gap look like a wave, maybe too orderly; it could suit some disciplined units.
- A unit can no longer be forced through a gap in the unit ahead of it to get it into the enemy line, and that is annoying. Cause: between two files 1.4 m apart a man has a comrade 0.7 m to either side, his crowd figure passes the limit and his drive is gone. Before the limit he kept part of his drive down to 0.82 m and squeezed through. The limit acts on every drive, a Move order's too. The report had said rear units would still force through where there is room; between files there is none, so in practice the limit stops them, which is close to the wait rule Gota rejected.
- Gota will try the 1.0 m body distance alone: `FL_SHOVE=0`.
- Gota: allied units making space for a unit that passes through them is needed in the near future. Noted in plan 020, track D.

## The record of 2026-10-04: what was tried on the packed mass, and the decision for now

Decision (Gota, 2026-10-04, night): keep the body distance at 1.0 m, without the shove limit. "Ok I prefer this one after all [`FL_SHOVE=0`]: each soldier trying to close the gap and looks more organic. Keep 1.0m". Gota may tune the 1.0 m value later; it is `BODY_DISTANCE` in `src/sim/soldier.rs`, and `FL_BODY=<metres>` sets it for a run (0.5 to 1.05, the wall's file pitch).

The session this record comes from: `/home/gota/.claude/projects/-home-gota-ggando-gamedev-flanks/bd33311e-ce4f-4632-87cd-124c50ea6591.jsonl` (Claude Fable 5.1, 2026-10-03 to 10-04). Gota expects to come back to it for the pushing and the soldiers' behaviour.

### In order

1. First play of stages 1 to 3. Gota: "It's better than when I first tried out unit scale 1.64 before this convo, but my first impression is that it still feels like pretty tight". Screenshots: `refs/unit_scale_branch/` (`looks_tight.png`, `looks_fine0.png` to `looks_fine2.png`, `debug0.png`, `debug1.png`, later `debug2.png`).
2. Where it is tight. Gota: "In 20k battle with no rear units in big grassland I don't see this issue. It looks like in 200k the soldiers from rear units trying to approach some unit on my side go through the first unit and soldiers from the rear push the ones in front while moving forward and that seems to be why it ends up being tight". The code agrees (section "What the code does").
3. The wait rule (617f057). Built on Gota's "let's try that first, though I'm not sure yet about what risks would it bring". Rejected after 20 seconds of play: "This doesn't look good AT ALL ... all the soldiers in rear look frozen and lifeless ... Trying to go around unit by pathfinding or forcing through front soldiers are better than this". Removed in 23722f1.
4. Paths around friendly units on the planned pathfinding. Rejected as a proposal: "don't rely on shit that doesn't exist on this branch, no excuses or laziness. we don't even know how heavy pathfinding is ... dont escape from this fix".
5. Pushing the line, Gota's idea: "I feel like it can be temporarily packed at 0.9m but the problem is that the rear soldiers pushing the soldiers toward frontline don't seem to propagate to the enemy side of frontline and defender's rear ranks. I think realistically, in this situation (attackers more soldiers than the defender and pushing physically), defenders would gradually be pushed away by the attacker". Written up as `docs/plans/023-pushing-the-line.md`. Gota: "023-pushing-the-line.md is great but sounds like a topic on another branch potentially".
6. Why 0.2.1 did not show it: the same distances, a smaller drawn man. Figure `work/notes/vis/032-same-press-two-sizes.png`.
7. The body distance. Gota: "Since we changed unit scaling, we can change it if it's reasonable", and "I feel it could be like 0.9 -> 1.0m to avoid overlap and then introduce the 1.15 or 1.2m thing". Built as a8dbdf6 and kept.
8. The shove limit (3ab4fc5). Gota's reading of it before the play: "you wanna go somewhere, you need to go through the crowd but since crowd might be fighting at frontline or whatever so rather than hitting ppl shoulder to shoulder you may or may not want to maintain some distance to maintain order". After the play: "with the shove limit, the soldiers moving over to close the gap make it look like a wave, maybe too orderly, though it could be a good behavior for some elegant units. But I can no longer force a unit to weave through a gap in the forward unit to forcefully let it into the enemy line and this can be very annoying." Removed in b87b77b; the default behaviour after the removal has the same fingerprints as `FL_SHOVE=0` had (direction and wide pile). If a disciplined unit type should move that way one day, 3ab4fc5 is the code to take it from.
9. Wanted soon, Gota: "we also need to create the 'make space for allied passing through units' behavior at some point in the near future". In plan 020, track D.

### What the kept setting measures

The 200k AI battle at the kept setting, one run (`tmp/runs/soldier-scale/body/ai200k-body10.log`): fighting units 1.16 m at 30 s, then 0.89 to 0.91 m from 60 to 100 s, the tightest unit 0.77 to 0.80 m, units on the move 0.89 to 0.92 m. Against 0.82 to 0.87 m and 0.79 to 0.83 m at 0.9 m. The picture at 105 s still shows a dense carpet in the centre of the mass, a little more ground than before. So the packed look at 200k is reduced a little, not solved; what would solve it is plan 023 (the line gives ground) or where an attack lays a unit's slots, and neither is on this branch yet.

Figure of the three settings side by side, the 200k battle and the six-on-one pile: `work/notes/vis/033-body-distance-and-shove-limit.png`. The green in its last pile picture is selection rings.

### Figures and pictures of this session

- `work/notes/vis/029-arc-and-spear-lane.png`: the arc of a cut and the two ways to aim a spear's lane.
- `work/notes/vis/031-contact-band.png`: the strip between two lines, the measured distances (the first version claimed a cause that was wrong).
- `work/notes/vis/032-same-press-two-sizes.png`: the same press drawn at 0.2.1's size and at true size.
- `work/notes/vis/033-body-distance-and-shove-limit.png`: the mass at 0.9 m, 1.0 m, and 1.0 m with the limit.
- Their scripts: `work/scripts/viz/arc_and_spear_lane_029.py`, `contact_band_031.py`, `same_press_two_sizes_032.py`, `body_distance_and_shove_limit_033.py`. The last two read screenshots under `tmp/runs/soldier-scale/`, which is disposable.
- Run output: `tmp/runs/soldier-scale/` (`s1/`, `s23/`, `arc/`, `sizes/`, `body/`, `room/`, and `roomrun.sh`).

### Switches on the branch after b87b77b

| Switch | What it does | Default |
|---|---|---|
| `FL_FIGHT_ROOM=0` | The fight as on main: closing to 1.2 m, no step back, no clear arc, body distance 0.9 m | on |
| `FL_BODY=<metres>` | The body distance | 1.0 |
| `FL_GAP=k,m,s,a` | Slot gap per kind in normal order | 1.4 each |
| `FL_UNIT_SCALE=<f>` | Scales the 1.70 m soldier; 0.588 draws 0.2.1's size | 1.0 |
| `FL_LOG_ROOM=1` | The room log every 5 s | off |
| `FL_DEBUG_ARC=1` or `=<metres>` | Draws the arc of the cut and the spear's lane under the soldiers near the camera | off |

Gone: `FL_WAIT_BEHIND` (with 617f057) and `FL_SHOVE` (with 3ab4fc5).

The cost of the switches: each is read from the environment once and kept (`OnceLock`), so the sim pays a load and a branch per use. The step time at 200k did not rise with them: 9.5 against 9.6 ms with the wait rule on and off, 8.5 to 8.6 ms at the kept setting against 8.9 to 9.1 ms before the body distance changed. The log systems (`room_log` and the nine older ones in `regiments.rs` and `formation.rs`) look their switch up in the environment every frame; that is small and could be read once.

## The state of the branch at b87b77b, and what remains (2026-10-04, night)

Gota: "I would say the this packed issue I observed earlier is solved for now." `FL_SHOVE` removed is fine.

The decisions at the start of the thread (2026-10-03), for the record: the height 1.70 m; the branch `feat/soldier-scale` off main; stages 1 to 3 back to back before the first play; `FL_GAP` for the gap per kind; the knight's pose wiring on its own branch off main; the first play does not wait for Astra's poses.

What the branch changes against main:

1. Every soldier is drawn 1.70 m tall, with the detail thresholds, the pose shader's feet, the 0.5 m ring and the arrows' heights following.
2. Rule 1: a man fights at 0.9 of his reach from his enemy and steps back to it.
3. Rule 2: no blow through a comrade in the arc of the cut or the lane of the thrust; he waits and sidesteps.
4. The body distance is 1.0 m.
5. `FL_GAP`, with 1.4 m for every kind by default.
6. Two tools: `FL_LOG_ROOM` and `FL_DEBUG_ARC`.

Twelve commits; two pairs cancel (617f057 with 23722f1, 3ab4fc5 with b87b77b). Not pushed, no PR.

What remains:

- The arc of rule 2. Gota, 2026-10-04: "requiring 70 deg cone for every attack animation is too restricting. each animation should have their own cone ... I think you could always stab even if you dont have 70deg clearance. If animation wise clearance is expensive, I'd loosen the clearance req to 50 or something." Claude's view, not built: the cost is no reason to loosen it, since the test runs in the quarter-second look only for men with an enemy in reach, and a second test in the same pass is one more comparison per neighbour. The sim already picks a style per blow, stab or chop, at random. The chop the shader plays is an overhead one, in a vertical plane, so 70 degrees to either side is far more than it sweeps; that width would fit the sideways slash, which is benched. Proposed: the blow follows the room. A stab needs only the lane to the enemy (0.35 m to either side, as the spear has), a chop a narrow arc, and a man picks among the blows he has room for and waits only when he has room for none. It also gives a crush its killing back.
- Killing is slower with rule 2 (11 to 51% more victims left in the two-on-one pile at 20 to 60 s, 25% more in the wide pile at 55 s), and in a crush under Move orders it nearly stops (4.8 times slower in the 200k scripted front). The stab above is the answer to both.
- The mass at 200k is still dense at the kept setting (0.89 to 0.91 m). Solved for now by Gota's word; the fix is plan 023 on another branch.
- Archers kill about 1.9 times as fast at true size. Not recalibrated.
- The charge's push on a line was measured only with the shove limit on. Not measured at the kept setting.
- Astra's narrow pose is not in. At 1.0 m a knight's arms and shield still overlap a neighbour's by up to 0.09 m in a press, and rear ranks still level swords into the man ahead.
- `FL_GAP` values are not chosen (bowmen wider), and the Demo layout assumes 1.4 m.
- The camera's lowest height, the walk and run speeds (plan 020 track C), and allied units making space for a unit passing through (track D) are untouched.
- Not retaken: the behaviour baselines, the fps baselines by the perf rules, the four benchmark views.
- The switches: `FL_FIGHT_ROOM` and the old code path go once Gota is sure of the two rules; the sort of all `FL_` switches before 0.3.0 is in plan 020.
- `tmp/runs/soldier-scale/target-main` (1.9 GB, main's baseline build) can go when the branch is merged.
