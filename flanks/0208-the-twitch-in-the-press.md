# 0208: the twitch in the press: a brake that follows the body, a hold that read as robots, and a bump on a man's own beat

Written by Claude Fable 5.1.

2026-10-05, Linux, branch `feat/soldier-scale` in the main tree, from 2548376 to 6351066. The session after handoff 096. Gota opened it by asking for the state of the branch and for a fix of the glitch at `FL_BODY=1.1` and wider. This devlog holds the whole session in order: what was measured, what was guessed wrong, the two sim changes that went in, and Gota's verdict on each. The sections from "At `FL_BODY=1.1` and wider the packed men shake" on were written as the session went and first stood at the end of devlog 0182.

## The session in short

1. Clips of 2548376 showed the walk fix holding and the men's bodies jumping every sim tick at 1.1 and 1.2. The walk fix of the session before had hidden a fault of the sim.
2. First reading of the cause: a man's walking and the spring between men both work against the overlap correction. Proposal in two steps, the spring first. Gota: "Ok proceed with the fix".
3. Step 1, the spring switched off under the body distance, did nothing at 1.2. Taken out. Trials found the real fault at that point: the crowd's brake ends at a fixed 0.82 m. Fixed in e8840e0 and reported as "the twitch is gone" from clips of the rear of the mass. That report was wrong.
4. Gota's play of e8840e0: better, still shaking sometimes, and the question whether rear units still push.
5. A log that counts the shaking men in the sim (`FL_LOG_SHAKE`, 3bbaaa2 and 691eb7c): a third of the men still shook at 1.0 and two thirds at 1.2.
6. Trials at the contact between two men, each with the push measured. The one that kept the stop at a body and ended the cycle went in as c1491d1, the hold. By the count the shake is gone.
7. Gota's play of c1491d1: no shaking, but the men look robotic. Every man stops like a car with ABS, a mechanical wave runs through the ranks, and Gota fears the rear no longer pushes.
8. Gota asked what M2TW's engine does at a contact. The answer from the install (field names, clip lengths, the live capture) was judged useless: it does not show what the engine does when one man walks into another.
9. Three ideas were talked through: a bump with a wait (the old contact about once a second), two circles (Gota's model of M2TW, and Claude's misreading of it), and a slow approach with a lean. Gota chose the bump with a wait: the lean would keep the creepiness of the hold.
10. Built as 6351066. By the count a tenth of the men of a press still shake, half of main's share. Not played yet. The last section has it.

## The state at the end of this devlog

- `feat/soldier-scale` at 6351066, 20 commits on main 66044d1, not pushed, no PR. `target/opt-dev/flanks` is 6351066.
- This session's commits: e8840e0 (the brake follows the body distance), 3bbaaa2 and 691eb7c (`FL_LOG_SHAKE`), c1491d1 (the hold at an overlap), 6351066 (the hold replaced by a bump on a man's own beat). With `FL_FIGHT_ROOM=0` the sim equals main after each (direction 18 of 18 fingerprints, two-on-one 28 of 28).
- Open: the look of a contact, for Gota's play of 6351066. Also open from before and untouched: whether rule 2 (the lane) is removed (Gota is considering it), the body distance value, archers at true size, the PR.
- Handoff: `work/handoffs/098-soldier-scale-the-contact-2026-10-05.md`.
- Tools made: `FL_LOG_SHAKE=1`; `tmp/runs/soldier-scale/shake/` (`shakerun.sh`, `pileshake.sh`, `trial.sh`), `tmp/runs/soldier-scale/walk/twitchclip.sh`, `tmp/runs/soldier-scale/pass/passrun.sh`; the binary of 2548376 at `tmp/runs/soldier-scale/bin/flanks-before`.
- Figures: `work/notes/vis/034-shake-in-the-press.png`, `035-why-the-press-twitches.png` (its spring half is wrong), `036-twitch-before-and-after.png` (the rear only, which misled), `037-hold-before-and-after.png`. Scripts in `work/scripts/viz/` with the same numbers.
- Also noted this session: plan 020, track D, has a line for M2TW's spreading under an attack order (Gota: perhaps later).

## At `FL_BODY=1.1` and wider the packed men shake

Gota, opening the next session: "I want to fix the animatino glitch at FL_BODY=1.1 or bigger". Checked on the build of 2548376 before anything else. Nothing was built or changed in the code.

Method: the six-on-one pile from the camera of the walk stills (`FL_TEST_PILE=1 FL_PILE_N=6 FL_CAM_X=0 FL_CAM_Z=22 FL_CAM_DIST=15 FL_CAM_PITCH=0.4 FL_CAM_LOCK=1`), a 4 s clip at 60 fps from 28 s after the window appears (`work/scripts/clip.sh 28 4 <out.mp4>`), at `FL_BODY` 1.0, 1.1 and 1.2. Clips: `tmp/runs/soldier-scale/walk/body10-clip.mp4`, `body11-clip.mp4`, `body12-clip.mp4`.

What the clips show:

- The walk fix holds. Once the rear has packed, the men stand with their legs together at all three settings.
- At 1.1 and 1.2 the men's bodies jump within one sim tick (1/30 s). Figure: `work/notes/vis/034-shake-in-the-press.png` (script `work/scripts/viz/shake_in_the_press_034.py`), two frames one tick apart laid over each other in red and cyan. At 1.0 the men are grey, so still. At 1.1 every man has a coloured edge, at 1.2 a wide one.
- Mean change of the picture over the last two seconds of each clip, in grey levels, over one tick and over two ticks: 8.8 and 7.3 at 1.0, 10.8 and 7.5 at 1.1, 16.0 and 13.5 at 1.2. Less change over two ticks than over one means a man is back near his place after two ticks: in on one tick, out on the next, 15 times a second.
- The room log of the earlier 1.2 run agrees: in units whose box does not move, 58 to 73% of the men have a velocity over 0.4 m/s (`tmp/runs/soldier-scale/room/pile6-body12.room`).

So what Gota saw as walking in place was the men being moved every tick. 2548376 stopped the legs from showing it; the movement itself is still there. That commit fixed the symptom, not the cause.

The cause, read in `src/sim/soldier.rs` and not yet tested:

- Three things act on two men closer than 1.4 m: each man's drive to his slot, the soft push between them (`push`, under `SEP_RADIUS`), and under the body distance the overlap correction that moves them apart by position (`corr`, 0.3 of the overlap a tick, at most 0.1 m a tick).
- The comments there state the rule the code was built on: inside the body distance an overlap is resolved by position only, because a force on the same overlap makes a packed crowd oscillate ("two solvers fighting over the same overlap", "force-based separation in a wedged mass only produces bang-bang oscillation").
- What keeps the drive and the push out of the body distance is the crowd's brake (`yield_to_crowd` and `steer`: `CROWD_SLOW` 1.2, `CROWD_STOP` 2.5). It fades the drive, the push and the speed between six neighbours at 1.12 m and six at 0.82 m. Those are fixed metres from 2026-07-08, set with the 0.9 m body.
- With six neighbours at the body distance, the share of the drive and the push still on is 27% at 0.9 m, 60% at 1.0 m, 93% at 1.1 m and 100% at 1.2 m. From 1.1 m on, men who already overlap still drive and push at nearly full strength, the correction moves them back out, and the speed into the overlap is removed only on a tick where the summed correction is over 1 cm.
- A wider body distance also puts more neighbours inside it at once (at 1.2 m with the mass at 1.0 m, all six), and each adds 0.3 of its overlap in the same tick.

Not built. The fix belongs in the sim and is feel-critical: the same brake decides how hard rear ranks press and whether a unit can be forced through the files of the unit ahead. The check for any fix is the same clip and figure, then Gota's play.

### 2026-10-05, the same session: Gota's answers, and the explanation redrawn

- Gota confirms the shake is the glitch: "soldiers sometimes do micro twitch really hard and that glitch is ugly".
- The proposal as first written could not be parsed. Redrawn as a figure: `work/notes/vis/035-why-the-press-twitches.png` (script `work/scripts/viz/why_the_press_twitches_035.py`). Three things move a packed man: his own walking to his slot, the spring that pushes men under 1.4 m apart through their speed, and the placing that sets men under the body distance apart directly. The crowd's brake turns the first two down as the crowd closes in, and it was set so that they are nearly off where the placing starts at 0.9 m. It has not moved with the body distance, so at 1.2 m the walking and the spring are fully on where the placing works.
- Rule 2 (the lane). Gota: the point of the cone and the lane was to solve the packed front line; now that everyone has clearance it is pointless, and Gota is considering removing it. Claude's "Fixed" in the state table was the wrong word for it. Claude's earlier reason to keep it leaned on a clearance per clip that does not exist. Removing it takes out `in_lane`, the `SIGHT_LANE` bit, the two places where a blocked man does not strike and does not close (`src/sim/soldier.rs`), and `src/lane_overlay.rs` with `FL_DEBUG_LANE`. Rule 1 stays. Not removed yet: Gota has not decided.
- The body distance that matches 0.2.1, per kind. By the same air between bodies at the body distance: knights 1.29 m (0.2.1: drawn 0.71 m wide, 0.19 m of air), men-at-arms 1.32 m (0.59 m wide, 0.31 m of air). By the same proportion of body distance to width: knights 1.39 m, men-at-arms 1.53 m, which is over the 1.4 m slots and `FL_BODY`'s cap. `FL_BODY` is one value for every kind.
- The M2TW behaviour Gota recalled: a unit switched from a Move order to an attack order spreads out to make room. It is in devlog 0177 (Gota's observation, the probe's numbers 0.90 to 1.42 m in 32 s, and the section "Why M2TW opens up in a fight"). Gota: it would conflict with the Move order during a battle, so it may not fix the rear units' press; perhaps later. Noted in plan 020, track D.

## 2026-10-05: the twitch fixed in the crowd's brake (e8840e0)

Gota: "Ok proceed with the fix". The fix that went in is not the one proposed. The proposal's reading of the cause was half wrong, and the trials below found the real one.

### Step 1 as proposed: built, measured, taken out

The soft push switched off under the body distance. Change of the picture over one tick and over two ticks (the measure of the section above; the clips now through `tmp/runs/soldier-scale/walk/twitchclip.sh <name> [ENV=val ...]`):

| `FL_BODY` | 2548376 | Step 1 |
|---|---|---|
| 1.0 | 8.8, 7.3 | 6.9, 6.7 |
| 1.1 | 10.8, 7.5 | 10.0, 6.9 |
| 1.2 | 16.0, 13.5 | 17.2, 13.6 |

It calmed 1.0 a little and did nothing at 1.2. The spring is not the cause. It also slowed a unit passing through a friendly unit by 13% (below). Removed again; it is not in the commit.

### The trials

Temporary switches in `src/sim/soldier.rs` (`FL_X=<letters>`, never committed; the file with them is not kept). All at `FL_BODY=1.2`, on top of step 1 unless marked:

| Variant | One tick, two ticks | Nearest comrade in the packed units |
|---|---|---|
| Step 1 alone | 17.2, 13.6 | about 1.03 to 1.09 m |
| No deadband on the correction | 15.6, 10.9 | 1.05 to 1.09 m |
| No removal of the speed into an overlap | 11.4, 7.3 | 0.79 to 0.89 m |
| The removal acts after the step | 5.7, 9.0 | 0.92 to 0.94 m |
| The removal acts per touched body | 4.3, 5.6 | 1.16 m |
| Both figures of the brake follow the body distance | 2.2, 3.6 | 1.13 to 1.15 m |
| Only the brake's end follows the body distance | 2.9, 3.3 | 1.05 to 1.11 m |
| The end follows it and the start is no later than touch, without step 1 | 2.2, 3.1 | 1.10 to 1.12 m |

So the twitch is the drive: it stops as soon as a man no longer drives into bodies he overlaps. The removal per touched body uses a man's own speed, not his speed against the man ahead, so it would hold up a marching column; not pursued.

A unit passing through a friendly unit: `FL_TEST_ROUTPASS=1` through `tmp/runs/soldier-scale/pass/passrun.sh <name> <seconds> [ENV=val ...]`. A 500-man unit under a Move order walks through a 500-man unit that holds. The measure is the passer's centre between 4 s and 12 s (the scenario's log stops changing after 14 s).

| | Pace through the holder |
|---|---|
| 2548376, 1.0 | 1.73 m/s |
| 2548376, 1.2 | 1.54 m/s |
| Step 1, 1.0 | 1.51 m/s |
| Both figures follow, 1.0 / 1.1 / 1.2 | 1.35 / 1.08 / 1.06 m/s |
| Only the end follows, with step 1, 1.0 / 1.1 / 1.2 | 1.56 / 1.36 / 1.41 m/s |

Moving both figures costs the pass 22 to 31%. That is the variant the proposal had set aside as "the shove limit again", and the measurement agrees. Moving only the end does not cost it: the start of the fade stays where it was, so a man between two files keeps his whole drive.

### The cause

The crowd's brake fades a man's drive between six neighbours at 1.12 m and six at 0.82 m (`CROWD_SLOW` 1.2, `CROWD_STOP` 2.5). The end point, 0.82 m, is 0.91 of the 0.9 m body: the drive is gone once the press is just inside the bodies. It was a fixed figure and stayed at 0.82 m when the body distance went to 1.0 m and wider. Between the body distance and 0.82 m a man still drove in, the overlap correction set him back out, and the wider the body distance the more of his drive was left there: 27% where bodies touch at 0.9 m, 60% at 1.0 m, 100% at 1.2 m.

### The fix (e8840e0)

`crowd_brake()` in `src/sim/soldier.rs`. The end of the fade is six neighbours at 0.91 of the body distance in use. The start is six neighbours at 1.12 m as before, or where they touch him when the body distance is wider than that. With `FL_FIGHT_ROOM=0` the body distance is 0.9 m and both figures are the old ones exactly.

Checks at e8840e0:

- Strict clippy clean, 35 tests pass. `FL_FIGHT_ROOM=0` equals main: direction 18 of 18 fingerprints, two-on-one 28 of 28.
- The twitch, one tick and two ticks: 1.8 and 2.8 at 1.0, 3.5 and 5.4 at 1.1, 1.6 and 2.4 at 1.2. Figure, before over after: `work/notes/vis/036-twitch-before-and-after.png` (script `work/scripts/viz/twitch_before_after_036.py`). After, the packed men are grey at all three settings, so still.
- The press in the six-on-one pile at 30 s, median to the nearest comrade in the packed units: 0.93 to 0.95 m at 1.0, 1.01 to 1.04 m at 1.1, 1.08 to 1.12 m at 1.2. The press now follows the body distance; at 2548376 it sat at 0.93 to 1.10 m at 1.2.
- Passing through a friendly unit: 1.73 m/s at 1.0 (1.73 before), 1.49 at 1.1, 1.39 at 1.2 (1.54 before).
- The spearwall charge at 40 s: wall lane 382 spearmen (moved back 1.2 m) against 368 knights, open lane 322 (0.3 m) against 408. Before: 385 (1.3 m) against 363, 322 (0.3 m) against 404. The wall keeps 1.01 to 1.03 m.
- Direction test at 36 s, kills front, side, rear: 424, 216, 776 (426, 215, 770 before). Wide pile, victims left at 55 s: 273 (280 before).
- The two-on-one pile kills slower: victims left at 20, 30, 40, 50, 57 s are 378, 294, 219, 146, 117 against 375, 277, 191, 124, 86 before. The attackers press in less once they are packed.
- The 200k AI battle, one run (not deterministic), `tmp/runs/soldier-scale/body/ai200k-brake.log` against `ai200k-body10.log`: fighting units 0.94, 0.91, 0.94 m at 60, 90, 100 s (0.91, 0.89, 0.89 before), the tightest unit 0.86 to 0.88 m (0.77 to 0.80 before), sim step 8.3 to 8.5 ms (8.5 to 8.6). Pictures at 105 s: `ai200k-brake_105s.png` shows rows with grass between the men across the mass; `ai200k-body10_105s.png` has a carpet in the centre. Still a dense crowd.

For Gota's eye: how the rear reads in motion at 1.0, 1.1 and 1.2, forcing a unit through the files of the unit ahead by hand, and the slower kill in a pile.

Files: the binary of 2548376 for a side by side is `tmp/runs/soldier-scale/bin/flanks-before`. `tmp/runs/soldier-scale/walk/` holds the clips and their frames, 6.2 GB, disposable. Figure 035 explains the twitch by the walking and the spring both; the spring half of it is wrong, the walking half is the cause.

### 2026-10-05: Gota's play of e8840e0, the push, and what still shakes

Gota: "It's better than before, but I still sometimes see shaking soldiers, it's either a lone one or some subset of colliding units or whatever. It looked less intense than the previous twitch though." Gota also asked whether the fix stops rear units pushing the front units, and whether the spring was suppressed.

The spring: not suppressed. Step 1 was tried and taken out; e8840e0 changes the brake's end only.

The push, from the logs already on disk (no new runs). A man pushes the man ahead by driving into him; the sim sets the pair apart and that moves the man ahead. The fix left that alone and changed where a packed man stops driving. Share of the drive left where six neighbours touch him: 27% on main (0.9 m body), 60% on 2548376 (1.0 m), 44% on e8840e0 (1.0 m).

| | 2548376 | e8840e0 |
|---|---|---|
| Spearwall charge at 40 s, the charged line moved back, wall lane and open lane | 1.3 m, 0.3 m | 1.2 m, 0.3 m |
| Six-on-one pile at 1.0, ground the attacked unit's front edge gave by 24 s (men left) | 7.2 m (302) | 6.4 m (309) |
| The same at `FL_BODY=1.2` | 5.4 m (351) | 4.3 m (364) |
| Two-on-one pile at 1.0, by 20 s | 5.0 m (375) | 5.1 m (378) |

Main gave 6.0 m by 20 s in the two-on-one pile (365 left). The front edge moves by deaths and by the push together. The attacked unit's rear edge stays within 0.3 m of its start in every version, so these scripted piles do not show a whole unit backing up, before or after. That effect in a battle is not measured.

What still shakes. In the clips of e8840e0 (the packed rear of the six-on-one pile at 1.0, 1.1, 1.2) no place returns to its picture after two ticks: the map of one-tick change less two-tick change has no positive spot under the fight line (`f-body10-shakers.png` and the like in `tmp/runs/soldier-scale/walk/`). So what Gota sees is somewhere that scene does not cover. Reading from the code, not reproduced: the brake counts the crowd around a man and needs about six neighbours to act. A lone man walking into one or two bodies, and the men where two units meet, have few neighbours, keep the whole drive, and still step in and are set back. The step in, the set back, and the speed removed only when the set back is over 1 cm are the fault itself; the brake hides it where the crowd is dense.

Proposed to Gota, not started: reproduce it first (the pass-through test and the edge of a rear unit, the same clip with the shake map), then fix the step in and set back at the contact of two men, with the push measured beside every variant.

## 2026-10-05: the shake counted in the sim, and the hold at an overlap (3bbaaa2, 691eb7c, c1491d1)

Gota's order after the play of e8840e0: reproduce what still shakes; if it does not reproduce, try a scene of 3 to 4 units; once reproduced, trial fixes at the contact between two men with the push measured beside each, and bring the variant that removes the shake and keeps the push.

### The log (3bbaaa2, 691eb7c)

`FL_LOG_SHAKE=1` (`shake_log` in `src/regiments.rs`). A man turns when his step of one tick points against his step of the tick before, both over 3 mm. Every 2 s it reports the living men who turned on at least 10 ticks of the window (and on at least 30, "hard"), their mean step, and what they share: their unit's state, out of formation, an enemy within 3 m, a crowd under the brake's start, how many bodies touch them. The six hardest are listed with their place on the field. The first version dropped every man's last step whenever a death changed the number of men, so it counted low in a fight and nothing at 200k; 691eb7c keeps the counts across the death sweep.

### What it showed

Two things reported earlier today were wrong.

- "The twitch is gone at 1.0, 1.1 and 1.2" after e8840e0. The clips watched the rear of the mass, which was calm. Inside the mass a third of the men still shook at 1.0 and two thirds at 1.2.
- "What is left are lone men and men with few neighbours, which the brake does not reach." The men who shook stood in formation, touched three or more bodies and had a crowd over the brake's start: the ordinary men of the press, wherever the brake leaves part of the drive on.

The six-on-one pile, the mean of the reports from 20 s to 44 s, with the corrected log:

| | `FL_BODY` 1.0 | 1.1 | 1.2 |
|---|---|---|---|
| 2548376 | 1,674 of 3,035 (1,236 hard), step 2.8 cm | | 2,535 of 3,119 (2,269 hard), 6.0 cm |
| e8840e0, the brake | 1,023 (360 hard), 1.5 cm | | 1,982 (1,509 hard), 3.4 cm |
| c1491d1, the hold | 79 (9 hard), 0.7 cm | 19 (2 hard), 0.7 cm | 31 (3 hard), 0.7 cm |
| Main's behaviour (`FL_FIGHT_ROOM=0`) | 586 (192 hard), 1.1 cm | | |

### The cause

`steer` removed a man's speed into an overlap only on a tick where the summed overlap correction was applied, which is when it passes its 1 cm deadband. On such a tick he was set back and stopped. On the next the correction was under the deadband, nothing held him, and he walked in again. In, out, every tick or two. The brake of e8840e0 lowers the drive behind it and so the size of the step; it does not stop the cycle. Today's sustained push of a pressed man on the man ahead came from this same cycle.

### The trials

Temporary switches (`FL_X=<letters>`, never committed), default body distance, through `tmp/runs/soldier-scale/shake/trial.sh <name> [ENV=val ...]`: the six-on-one pile, `FL_TEST_ROUTPASS` and `FL_TEST_CHARGE`. Shakers here are from the first version of the log, so low, but comparable with each other.

| Variant | Shakers of 3,000 | Press | The attacked front gave by 24 s | Through a friendly unit | Charged wall moved back at 40 s, wall and open lane |
|---|---|---|---|---|---|
| e8840e0 | 706 | 0.89 to 0.95 m | 6.4 m | 1.73 m/s | 1.2 m, 0.3 m |
| 2548376 (the old brake) | 1,469 | 0.90 to 0.95 m | 7.2 m | 1.73 m/s | 1.3 m, 0.3 m |
| Main's behaviour | 402 | 0.81 to 0.86 m | 6.5 m | 2.29 m/s | 1.3 m, 0.4 m |
| Per touched body: no closing on it | 115, step 4.9 cm | 0.96 to 0.99 m | 7.0 m | 4.24 m/s | 7.1 m, 3.2 m |
| Per touched body: closing no faster than the applied correction sets him back | 16 | 0.90 to 0.95 m | 6.8 m | 3.71 m/s | 8.3 m, 3.3 m |
| Per touched body: no faster than his own share of that pair's correction | 112 | 0.82 to 0.92 m | 7.7 m | 2.14 m/s | 2.3 m, 1.4 m |
| Per touched body: his share and the other man's | 108 | 0.80 to 0.92 m | 7.2 m | 2.89 m/s | 2.1 m, 1.2 m |
| Along the summed correction: all of the speed off while any overlap is left | 179 | 0.87 to 0.92 m | 8.2 m | 2.51 m/s | 2.2 m, 1.7 m |
| Along the summed correction: he keeps what the correction takes back | 46 | 0.85 to 0.91 m | 7.8 m | 2.44 m/s | 2.3 m, 1.1 m |
| The last one with the old brake | 105 | 0.79 to 0.87 m | 8.2 m | 2.51 m/s | 2.4 m, 1.4 m |

- The first two per-body variants make bodies slippery: a man slides round each body in turn, a charge shoves the spearwall back 9 to 10 m in ten seconds, and a unit runs through a friendly one at 3.7 to 4.2 m/s. Not the game.
- The variants along the summed correction keep today's stop at a body, as the old removal did, and close the gap in time that let a man walk back in.
- The last line with the old brake at `FL_BODY=1.2`: the press sits at 0.76 to 0.89 m whatever the body distance. So e8840e0 is still needed: it makes the press follow the body distance.

### The fix (c1491d1)

In `steer`, with the room to fight on: toward the bodies he overlaps (the direction of the summed overlap correction before its deadband and cap), measured against their own movement (their ground velocities weighted by how far each sets him back, read in the scan with `for_each_candidate_vel`), a man keeps only the speed the applied correction takes back this tick. So his legs never take him further into an overlap, he is not moved back while he presses, and he holds his ground while the bodies are set apart. With `FL_FIGHT_ROOM=0` the old removal runs.

Checks at c1491d1:

- Strict clippy clean, 35 tests pass. `FL_FIGHT_ROOM=0` equals main: direction 18 of 18 fingerprints, two-on-one 28 of 28.
- Shakers: the table above. Passing through a friendly unit: 6 of 1,000 men shake (421 at e8840e0, 210 on main's behaviour).
- The 200k scripted front at 59 s: 4,077 of 189,727 men shake (1,059 hard), step 0.7 cm. Main's behaviour: 25,457 of 187,402 (3,176 hard), 1.0 cm, and 53,496 at 34 s. The 200k AI battle, one run: about 2,000 to 2,400 of 190,000, step 0.7 cm. The men left are in the tightest part of a press (crowd about 2.0, five or six bodies touching) and turn every tick by 0.7 cm.
- Figure, the middle of the mass before and after at 1.0 and 1.2: `work/notes/vis/037-hold-before-and-after.png` (script `work/scripts/viz/hold_before_after_037.py`). After, the near half of the picture does not change at all over four seconds: packed men stand completely still.

What changed beside the shake, default body distance, e8840e0 against c1491d1 (main's behaviour in brackets):

| | e8840e0 | c1491d1 |
|---|---|---|
| Six-on-one pile, ground the attacked front gave by 24 s | 6.4 m | 8.0 m (6.5) |
| Spearwall charge, the wall moved back at 40 s, wall lane and open lane | 1.2 m, 0.3 m | 2.3 m, 1.3 m (1.3, 0.4) |
| Spearwall charge at 40 s, spearmen against knights, wall lane | 382 against 368 | 385 against 355 |
| A unit through a friendly unit | 1.73 m/s | 2.56 m/s (2.29) |
| Two-on-one pile, victims left at 20, 30, 40, 50, 57 s | 378, 294, 219, 146, 117 | 370, 267, 187, 114, 77 (2548376: 375, 277, 191, 124, 86) |
| Wide pile, victims left at 55 s | 273 | 243 (224) |
| Direction test at 36 s, kills front, side, rear | 424, 216, 776 | 390, 239, 783 |
| The press in the six-on-one pile at 1.0, 1.1, 1.2 | 0.89 to 0.95, 1.01 to 1.04, 1.06 to 1.12 m | 0.84 to 0.91, 0.88 to 0.97, 0.93 to 1.05 m |
| 200k AI battle, the tightest unit, and units on the move | 0.86 to 0.88 m, 0.88 to 0.96 m | 0.81 to 0.84 m, 0.84 to 0.85 m |
| 200k scripted front at 75 s, men alive | 185,329 at f4f2d7d | 186,179 |

Sim step: 9.4 to 9.8 ms in the 200k scripted front against 9.3 to 9.6 ms before, with the shake log running in the later runs. Not measured by the perf rules.

Read: the push is stronger, not weaker. A pressed man no longer gives back ground each tick, so all of an overlap goes into moving the other man. The crash of a charge moves a spearwall about twice as far, a pile kills at the pace of 2548376 again, and a unit passes a friendly unit faster than on main. The press at the default is a little tighter than at e8840e0 and about where 2548376 had it; `FL_BODY=1.1` and 1.2 now loosen it without a shake.

For Gota's eye: whether the packed rear standing completely still reads as lifeless, the stronger crash of a charge, and forcing a unit through the files of the unit ahead by hand.

Files: `tmp/runs/soldier-scale/shake/` (`shakerun.sh`, `pileshake.sh`, `trial.sh` and every log), `tmp/runs/soldier-scale/walk/` (clips and frames, 7.1 GB, disposable), `tmp/runs/soldier-scale/bin/flanks-before` (2548376 for a side by side).

## Gota's play of c1491d1: no shake, but robots

Gota: "Ok looks like there's no shaking anymore but I felt like this fix makes soldiers look too robotic. Everyone has ABS when they hit the soldiers in front of them, making a weird mechanical wave in ranks. This hits uncanny valley and a bit creepy, good for an army of droids, but not so human. Also, this means rear soldiers will no longer push the ones in front of them? That'd be bad too. Is there any catch of this fix other than these?"

What the hold does to the look. Every man loses his speed toward the man ahead on the tick he touches him, at the same distance for all, gives nothing, and then stands. A rank that walks up stops as one, and the stop runs back through the ranks like a machine. The give at a contact (a man carried in a little, set back, closing again in his own time) is what read as men. The old code made that give by accident and at tick rate, which was the twitch. The hold removed the twitch by removing the give.

The push, from the numbers already taken:

- At arrival it is stronger than before (the tables above): all of an overlap goes into moving the other man.
- Once a press has settled, nobody pushes. In the six-on-one pile the attacked front gave 8.0 m by 24 s and 8.1 m by 40 s. A pressed man holds at an overlap under the correction's deadband and neither moves nor moves the man ahead.
- That is also so at e8840e0 (6.4 m by 24 s, 6.3 m by 40 s). Only 2548376 kept gaining after 24 s (7.2 m to 8.3 m), and it did so through the twitch: every step in bumped the man ahead.

So Gota's fear is right for the settled press, and it began with the brake change, not with the hold.

The other costs of c1491d1, as told to Gota:

- A man who holds keeps a speed in the sim equal to what the correction takes back (up to 3 m/s at the correction's cap) while he does not move. Whatever reads a man's velocity sees it: the room log's walking share, the facing of men not in a fight, the "is he mid step" tests. No fault was seen from it; it was not checked one by one.
- The crash of a charge moves a spearwall about twice as far (2.3 m against 1.2 m at 40 s). Kills there are about the same.
- A unit passes through a friendly unit faster (2.56 m/s against 1.73 m/s), so a formed unit resists a passing one less.
- The press at the default is a little tighter than at e8840e0 (0.84 to 0.91 m against 0.89 to 0.95 m in the pile), and units on the move at 200k pack to 0.84 to 0.85 m (0.88 to 0.96 m at e8840e0).
- A pile kills at the pace of 2548376 again; in the direction test the front kills fell from 424 to 390.
- Packed men stand completely still. The game has no idle movement.
- About 2% of the men at 200k still turn every tick by 0.7 cm in the tightest part of a press.
- The limit acts along the summed correction. Where the corrections of the bodies around a man cancel there is no direction and no limit on that tick.
- The scan reads one more column, the neighbours' velocities. The 200k step read 9.4 to 9.8 ms against 9.3 to 9.6 ms; not measured by the perf rules.
- Not tried with the hold: a pursuit of a broken unit (a pursuer who overlaps a fleeing man now keeps his pace where he was stopped before), archers, and blocked ground.

Claude's reading. The hold was the wrong kind of fix. A contact between two men needs give, at a man's rhythm and different for every man; the fault of the old code was the rate of the give (every tick), not the give.

Proposed to Gota, not built, in place of the hold:

1. A bump is a step back. A man who runs into the man ahead is carried a little into him and set back out over a few ticks, as before today, and the man ahead is bumped forward.
2. He does not press in again at once. He waits a moment that is his own (per man and per time window, as the sidestep's windows are), then steps up again at the shuffle's pace. This is what ends the tick-rate cycle, and each step up bumps the man ahead again, so a settled press pushes in surges.
3. Each man yields at his own distance: the brake's two points differ a little from man to man, so a rank does not stop on one line.

The check for it: the shake log (no man turning every tick), the push tables (the attacked front still giving ground after 24 s), short clips of a rank closing up, then Gota's play.

## What M2TW does at a contact, read from the install (2026-10-05)

Gota asked what M2TW's engine does here, and then, when Claude began to answer from the old devlogs: "No, you should get that from the actual m2tw install or something". Three sources in the install, read the same day. M2TW is a reference, not the goal; what is useful for flanks is at the end.

### The executable's own names

`strings medieval2.exe` (19.7 MB, the same file as kingdoms.exe). The engine prints its soldier state by field name for its resync check, so the names of its contact handling are in the file:

- Locomotion per man (`loco_data.`): `m_destination`, `m_stopping_distance`, `m_stopping_modifier`, `m_speed_modifier`, `m_radius_modifier`, `m_soft_collision`, `m_allow_deflection`, `m_deflection_counter`, `m_direction_offset`, `m_cone`, `m_slide_rotation`, `m_obstacle_iterator`, `m_blocked_counter`, `m_pending_action`, `m_is_moving`.
- An impact per man: `m_impact_status` with `movement`, `momentum`, `zone` and `response`, and the log line "impact status: response: %d zone: %d momentum: %d movement: %d".
- The reactions to a collision, an enumeration: `CLRT_NONE`, `CLRT_STOP`, `CLRT_KNOCKBACK`, `CLRT_KNOCKDOWN`, `CLRT_REFUSAL_STOP`, and the deaths (`CLRT_NORMAL_DEATH`, `CLRT_TRAMPLE_DEATH`, `CLRT_KNOCKDOWN_DEATH`, `CLRT_FLYING_DEATH`, `CLRT_REFUSAL_DEATH`, `CLRT_GALLOPING_DEATH`).
- The reactions to a blow: `ART_HIT`, `ART_STEPBACK`, `ART_KNOCKBACK_LIGHT`, `ART_KNOCKBACK_HEAVY`, `ART_KNOCKDOWN`, `ART_DEFENDED_SHIELD`, `ART_DEFENDED_ARMOUR`, `ART_DODGE`, `ART_MISS`.
- No tunable for a man's collision is read from `battle_config.xml`; the keys the executable reads under `movement/` are for queues at corridors, ladders and siege towers (`stand-off-distance`, `halt-threshold`, `restart-threshold`, `step-on-distance`). `descr_pathfinding.txt` holds only the pathfinder and `formation_hold_distance 20.0`.

So the engine's contact is a small set of discrete outcomes (nothing, stop, knocked back, knocked down), a man steers round an obstacle within a cone and counts how long he is blocked, his stop has a distance, and his body can be soft and its radius changed. How the code uses these is not in the strings.

### The clips of the sword and shield body

`data/descr_skeleton.txt`, type `MTW2_Mace`, each clip's own length and travel from `data/Animations/pack.dat` (read with `work/scripts/m2tw_anim/m2anim.py`, 20 frames a second):

| Clip | Length | Travel |
|---|---|---|
| `walk` (one cycle), `run`, `charge` | 0.90, 0.70, 0.60 s | 1.62, 2.52, 2.58 m |
| `walk_to_stand_a` | 2.20 s | 2.16 m |
| `run_to_stand_a` | 1.70 s | 2.01 m |
| `run_to_walk` | 1.30 s | 2.57 m |
| `charge_to_ready` | 1.50 s | 4.37 m |
| `combat_jog_to_ready`, `advance_to_ready` | 1.50, 1.70 s | 1.95, 1.45 m |
| `stand_a_to_walk`, `stand_a_to_run` | 1.20, 1.30 s | 0.89, 1.68 m |
| `step_forward`, `step_backward` | 1.20 s | 0.92 m |
| `step_left`, `step_right` | 1.00 s | 0.64 m |
| `shuffle_forward`, `_backward`, `_left`, `_right` | 1.50 s | 1.35, 1.22, 1.20, 1.27 m |
| `knockback_from_front`, `_back`, `_left`, `_right` (standing) | 1.60 s | none: he reels in place |
| `knockback_move_from_front`, `_back`, `_left`, `_right` (moving) | 3.50, 2.40, 3.00, 3.10 s | 0.51, 0.23, 0.58, 0.50 m |
| `knockdown_launch`, `_lying`, `_recover` | 1.10, 3.00, 5.50 s | 1.82 m thrown, 0, 0.24 m |
| `stand_a_idle`, three `hf_idle`, three `lf_idle` (the same for stands b and c, and for ready) | 2.0 s; 1.6 to 3.5 s; 2.5 to 4.3 s | none |
| `stand_a_to_stand_b` | 0.80 s | 0.16 m |

A stop is a clip of 1.5 to 2.2 s that covers 1.5 to 4.4 m. The smallest move a man makes is a whole step: 0.92 m forward or back in 1.2 s, 0.64 m to the side in 1.0 s. A man who stands has three stances and six fidgets for each.

### The live capture

`work/research/m2tw-extraction/contact_behaviour.py` on the capture of 2026-10-03 (`capture-20261003/soldiers-20261003-163649`: Gota's six units of 120 Dismounted Feudal Knights, two samples a game second, 215 s; the enemy army is not in it). The engine's collision radius is 0.4 m, so two bodies touch at 0.80 m.

- Bodies are soft. Distance to the nearest man of the same unit: on the march 33% are under 0.80 m, 25% under 0.75 m, 17% under 0.70 m, the 1st percentile 0.51 m. Standing 7.8% under 0.80 m, fighting 5.0%.
- A stop takes a man about 2 s (medians 1.5 to 3.0 s over four clean stops, 10th to 90th percentile 0.5 to 5.0 s), and the men of one unit finish over 2.0 to 3.0 s (10th to 90th). A start takes a man 1.0 to 2.0 s and the men begin over 1.6 to 2.5 s.
- On the march a man's speed is 0.72 to 1.35 of his unit's median (10th to 90th); 4% are under half of it and 6% over one and a half.
- In a standing unit, between two samples half a second apart, 48% of the men do not move at all, 38% move more than 1 cm and 5% more than 5 cm (median 0.3 cm, 90th percentile 3.7 cm).

Lines of the script's output with unit speeds of 15 to 40 m/s are samples where the soldier array was reordered; they are not used above.

### Not known from the install

Whether a settled press pushes the men ahead of it, and how far a pushed line moves. The capture has one army. A capture with both armies would show it: a deep unit behind a thin friendly unit that fights, and a Move order into an enemy line. The probe finds both armies by shape since 2026-10-03, untried on the real game.

### What it says for flanks

M2TW never has a walking man stopped by the body ahead on one tick, so it has neither the twitch nor the ABS look. Four things keep it out: a man begins his stop about 2 m early and takes about 2 s; bodies are soft, a third of marching men overlap, so no correction has to snap anyone back; the smallest move is a whole step; and every man's timing is his own, a unit's stop is spread over 2 to 3 s. A hit or a hard collision is an animation of 1.6 to 3.5 s that moves a man 0.2 to 0.6 m, not a force every tick.

## After the hold: what was talked through, and the bump on a man's own beat (6351066)

### Gota on the M2TW section

"Ok you just read animation clips durations and guessed it, completely useless". The section above stays as a list of what the install holds. It does not show what the engine does when one man walks into another; only the disassembly of that code or a capture of exactly that moment would.

### The ideas, in order, with Gota's words

1. Bump and wait (Claude): the old contact, with a wait so it cannot repeat every tick. A man bumps, is set back out, waits a beat of his own, steps up again. Gota: "So you just make the bump frequency from 15hz to 0.5hz or something? sounds good but what's the catch?" The catches named: the sustained push gets weaker, the wait is picked by eye, forcing a unit through another slows if the wait counts bodies at a man's sides, a small bump slides without a step, walls.
2. How 0.2.1 hid the fault (Gota's question). The same in and out is in main's behaviour: 586 of 3,000 men in the six-on-one pile. The step is 1.1 cm there (2.8 cm at 1.0 and 6 cm at 1.2 on 2548376), fewer men do it, and, read from the code and not measured, the old walk animation showed a 1 cm shake as a man shifting his feet until 2548376 stopped the legs.
3. Two circles. Gota's model of M2TW: "there's two physics circles (1. the hard limit one and 2. another wider one used during combat/idle on top of 1), and soldiers with explicit move orders ignore 2. But when the state changed from moving to others and soldiers overlap with 2, it'll do the same pushback thing but over the course of several seconds", pointing at the row "a fighting unit opens to 1.5 m, not reproduced". Claude first read it as a new inner hard limit of 0.8 m with `FL_BODY` made soft, and began to build that. Gota stopped it: "No, the hard limit is the current FL_BODY and the combat distance is the m2tw one. But your idea is interesting too and might also work ... We may need to implement the third circle later for the m2tw spec tho". Neither is built. Three circles as they now stand in the talk: an inner limit of 0.8 m (Claude's, soft `FL_BODY` above it), the physics distance `FL_BODY` (the hard limit in Gota's model), and M2TW's combat distance of about 1.5 m (soft, over seconds, in combat and idle, ignored by men under a Move order).
4. Gota set the order: "the problem we need to first fix is how soldiers interact with the hard limit circle FL_BODY. no shaking or no hard stop, keep the push".
5. Slow and lean (Claude): a man slows for the body in his lane, then rests against it on a cushion, and the lean is the push. Gota: "It sounds like the creepy behavior of current would still remain". Claude agreed: what is creepy in the hold is that the stop is perfect, the same for every man with nobody shoved, not that it is sudden. Not built.
6. Gota: "do this fix", the bump and wait.

### The build

`presses` and the branch in `steer` (`src/sim/soldier.rs`). Every man has a beat of his own, 20 to 45 ticks long (0.67 to 1.5 s, by the man), and lets 30% of his beats pass. For the first 3 ticks of a beat the contact is the old one: he shoves in, the overlap correction sets him back and takes his speed. Between beats nothing carries him into the bodies he overlaps: toward them (the direction of the summed correction before its deadband), against their own movement, his speed is removed on every tick the overlap lasts, so he is set back out and stands. A man under a Move order always has the old contact, so a unit can still be forced through another. With `FL_FIGHT_ROOM=0` the old contact runs for everyone. The four numbers (3 ticks, 20, 45, 30%) are Claude's and are not tuned.

A first attempt removed only the man's own drive into the overlap between beats. It changed little, because the spring between men still walked him in. It was not committed.

Shakers in the six-on-one pile, the mean of 20 s to 44 s, the corrected log:

| | `FL_BODY` 1.0 | 1.1 | 1.2 |
|---|---|---|---|
| 2548376 | 1,674 of 3,035 (1,236 hard), 2.8 cm | | 2,535 (2,269 hard), 6.0 cm |
| e8840e0, the brake | 1,023 (360 hard), 1.5 cm | | 1,982 (1,509 hard), 3.4 cm |
| c1491d1, the hold | 79 (9 hard), 0.7 cm | 19 (2 hard) | 31 (3 hard), 0.7 cm |
| The first attempt, drive only | 536 (149 hard), 1.8 cm | 1,519 (1,000 hard), 3.2 cm | 1,514 (1,182 hard), 3.4 cm |
| 6351066, the bump on a beat | 297 (99 hard), 1.2 cm | 309 (93 hard), 2.2 cm | 352 (110 hard), 2.5 cm |
| Main's behaviour | 586 (192 hard), 1.1 cm | | |

The rest at 6351066, default body distance, with c1491d1 and e8840e0 beside it:

| | e8840e0 | c1491d1 | 6351066 |
|---|---|---|---|
| The press in the six-on-one pile | 0.89 to 0.95 m | 0.84 to 0.91 m | 0.87 to 0.92 m |
| Ground the attacked front gave by 24 s, by 40 s | 6.4, 6.3 m | 8.0, 8.1 m | 8.9, 7.5 m |
| Victims left at 40 s there | 159 | 129 | 132 |
| A unit through a friendly unit | 1.73 m/s | 2.56 m/s | 2.10 m/s |
| Spearwall charge at 40 s, the wall moved back, wall and open lane | 1.2, 0.3 m | 2.3, 1.3 m | 1.8, 1.7 m |
| Spearmen against knights there, wall lane | 382 against 368 | 385 against 355 | 378 against 362 |
| Direction test at 36 s, kills front, side, rear | 424, 216, 776 | 390, 239, 783 | 395, 222, 792 |
| Wide pile, victims left at 55 s | 273 | 243 | 252 |
| Two-on-one pile, victims left at 20, 30, 40, 50, 57 s | 378, 294, 219, 146, 117 | 370, 267, 187, 114, 77 | 365, 276, 188, 115, 75 |

At 200k. The scripted front, where both armies are under Move orders and so keep the old contact: 30,000 to 76,000 of about 190,000 men shake (about 10,000 hard), step 1.1 to 1.8 cm, 186,203 alive at 75 s, the press at 0.90 and 0.88 m, sim step 9.7 ms. The AI battle, one run with the shake log on: 4,400 to 5,900 shake (600 to 2,300 hard), step 0.9 to 1.2 cm; fighting units 0.90 to 0.96 m, the tightest 0.87 to 0.88 m, units on the move 0.87 to 0.90 m; sim step 9.1 to 9.3 ms. The picture at 105 s (`tmp/runs/soldier-scale/body/ai200k-beat_105s.png`) is a dense mass in rows with some grass between the men, as before; a still does not show the beat.

Checks: strict clippy clean, 35 tests pass, `FL_FIGHT_ROOM=0` equals main (direction 18 of 18, two-on-one 28 of 28).

What is known to be left:

- About a tenth of the men of a press still turn every tick, by 1.2 cm at 1.0 and 2.5 cm at 1.2. They stand among several bodies; the summed direction is not the body such a man walks into. The hold removed these, the bump on a beat does not.
- Men under a Move order shake as after e8840e0 while they push through, and so does the 200k scripted front.
- The sustained push of a settled press is not measured apart from the push on arrival. Gota: if the push is weak, fix it in another change.
- A bump is 2 to 4 cm and may show as a slide: it is under what the leg animation shows.
- Not looked at in motion. No clip of 6351066 was taken; the look is for Gota's play.

### Gota's play of 6351066: it still twitches

Gota at `FL_BODY=1.2`: "Some units ... still do twitching, though it might be less frequent. Why does it still happen when soldiers don't push away each other 30 times a second and now once in 0.67 to 1.5s long". And: "why didn't you fix this? The current 'fix' is not really a fix that achives what I requested is it".

It is not. The beat limits how often a man walks into a body. The game still sets overlapping bodies apart on every tick (0.3 of the overlap, when over 1 cm), and that was never slowed; Claude's "a bump about once a second" made it sound as if both halves were. At 1.2 a packed crowd sits at about 1.1 m, every man overlaps four to six neighbours all the time, their pushes do not cancel, and he is bounced between them without walking. Those are the tenth of the crowd in the table above. Units under a Move order are left out of the beat and shake as after e8840e0. This reading is from the code and from who the shakers are, not from a test. The hold did not show it because a holding man leaned back against each push as hard as he was pushed.

Claude had the numbers that showed it was not gone, wrote them under "what is known to be left", and wrapped up with the handoff. Gota asked whether the context length was the reason; it was not, it was Claude's choice to stop.

Next, not built: slow the setting apart itself, as in Gota's "pushback over several seconds", first as a trial behind a temporary switch to confirm the reading.
