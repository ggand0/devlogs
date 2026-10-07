# 0210: a port of the contact rules, what every version does at a contact, and the proposal that a contact shares the speed

Written by Claude Fable 5.1.

2026-10-06, Linux, branch `feat/soldier-scale` in the main tree at 6351066. The session after handoff 098. No game code changed, nothing was built and no game was run. This devlog holds the analysis and a proposal that waits for Gota.

## What Gota asked

"This is like the third thread trying to fix the unit spacing and I desire to completely fix everything in this session. So I want to make FL_BODY=1.2 work." Some soldiers still twitch hard at 6351066. No hard brake like ABS when a man is blocked. Keep the push when a unit is forced through the unit ahead by a Move order. Asked for: the situation described intuitively, the best fix proposed, and a picture of every major version of the spacing logic (0.2.1 included) with the proposal beside them.

## The port

`work/scripts/viz/contact_toy.py` ports the contact part of one soldier's tick from `src/sim/soldier.rs` to Python: `drive` (the walk to a slot, the hold deadzone and the step), `scan` (spring, overlap correction with its mass weights, crowd, the velocity of the overlapped bodies), `yield_to_crowd` (deadband, caps, the brake), `steer` (grip, acceleration cap, damping, the contact rule) and `integrate`. The contact rule and the brake of each version can be chosen: 0.2.1, 2548376, e8840e0, c1491d1 (the hold), 6351066 (the beat, with `presses` and the Move order exception ported exactly) and the proposal. The shake log's measure is ported too (a turn is a step against the step before, both over 3 mm; 10 turns in 2 s make a shaker, 30 a hard one).

It has no fight, no deaths, no footwork around an enemy, no stagger, no walls and no charge. So it shows a press and a contact, not a battle.

Scenes: `pile` (six units of 150 attack-ordered onto one that holds, laid out as `FL_TEST_PILE`), `pass_through` (`FL_TEST_ROUTPASS`: 34 by 15 men sent through 34 by 15 who hold), `front` (two lines of three units under Move orders, head-on), `file_meets_file`. `python3 work/scripts/viz/contact_toy.py` prints the tables below in about two minutes; the last output is `tmp/runs/soldier-scale/contact-toy/tables.txt`.

One trap: without the 0.25 m spawn jitter of `spawn_regiment` the files of two units line up exactly, nobody slips between files, and the pass-through reads 0.5 m/s instead of 1.6. The port has the jitter.

### How close it is to the game

Share of the men of the pile who shake (the game's counts from devlog 0208, of about 3,000 to 3,100; the port's of 1,050):

| | Game, 1.0 | Port, 1.0 | Game, 1.2 | Port, 1.2 |
|---|---|---|---|---|
| Main's behaviour (0.9 m body) | 20%, step 1.1 cm | 24%, 1.0 cm | | |
| 2548376 | 55%, 2.8 cm | 63%, 1.1 cm | 81%, 6.0 cm | 90%, 3.8 cm |
| e8840e0 | 34%, 1.5 cm | 39%, 1.0 cm | 64%, 3.4 cm | 67%, 1.8 cm |
| c1491d1, the hold | 2.6%, 0.7 cm | 9%, 0.7 cm | 1%, 0.7 cm | 10%, 0.7 cm |
| 6351066, the beat | 10%, 1.2 cm | 12%, 0.8 cm | 11%, 2.5 cm | 42%, 1.4 cm |

Pace of a unit under a Move order through a unit that holds, m/s:

| | Game | Port |
|---|---|---|
| Main's behaviour | 2.29 | 2.33 |
| 2548376, 1.0 / 1.2 | 1.73 / 1.54 | 1.61 / 1.61 |
| e8840e0, 1.0 / 1.2 | 1.73 / 1.39 | 1.58 / 1.53 |
| c1491d1, 1.0 | 2.56 | 2.40 |
| 6351066, 1.0 | 2.10 | 1.88 |

The press sits at the same distances (main 0.83 m against the game's 0.81 to 0.86; e8840e0 at 1.2: 1.07 against 1.06 to 1.12). The order of the versions is the game's in both tables. The port overstates the shake of the hold and of the beat at 1.2: nobody dies in it, so the pile never thins.

## What the port and the code show

1. In every version that was built, a running man loses all his speed on the tick he touches a body. In 0.2.1, 2548376 and e8840e0 he is then set back out and walks in again. The in and out is in 0.2.1 as well (a fifth of the men of the game's pile, in steps of 1 cm); it was small between men drawn 0.59 m wide.
2. The hold (c1491d1) and the beat between beats (6351066) take away a man's closing speed measured against the speed of the bodies he overlaps. Each of the two men of a contact does that alone and takes away all of it. So the runner drops to the speed of the standing man (zero), and the standing man is given the runner's whole speed on the same tick. The two swap speeds. In the port a man hit by a runner at 9.7 m/s moves at 9.3 m/s for one tick, then loses it to the next man. This is in the Rust code (`steer`: `into = (new_v - overlap_vel).dot(-dir)`, `new_v += dir * into` for both men). It fits Gota's words for c1491d1: every man stops as if he had ABS, and a mechanical wave runs through the ranks. Devlog 0208 read the stronger push of the hold as "all of an overlap goes into moving the other man"; the swap is the mechanism. Not seen in the game by Claude; read from the code and the port.
3. The beat (6351066): for 3 ticks of each beat a man has the contact of 0.2.1 again, one in and out about once a second per man. That is a twitch by construction, and with a 1.2 m body every packed man does it. Men under a Move order have the contact of 0.2.1 on every tick.
4. The setting apart moves a man by 0.3 of each overlap a tick. A packed man at `FL_BODY=1.2` overlaps six bodies, and six times 0.3 is more than one: a push from one side throws him past the middle of his neighbours, and he comes back on the next tick. The 1 cm deadband on the summed correction makes it come in jumps.

### Which change removes the shake

Shakers of 1,050 in the port's pile, at 1.0 and at 1.2 (from 20 s to 40 s):

| Variant | 1.0 | 1.2 | Note |
|---|---|---|---|
| e8840e0 as it is | 414 | 704 | |
| e8840e0, setting apart at 0.1 | 94 | 141 | the press tightens to 0.81 and 0.97 m |
| e8840e0, no deadband | 718 | 910 | worse: the legs are cut every tick, the setting apart overshoots every tick |
| e8840e0, 0.1 and no deadband | 110 | 324 | |
| The hold, 0.1 and no deadband | 2 | 2 | still the swap of speeds |
| Legs fade as he sinks into a body (give 0.15 m), setting apart 0.3, no deadband | 0 | 0 | still the swap |
| The same with the deadband | 30 | 54 | |
| Shared speed, setting apart 0.3, no deadband | 5 | 13 | |
| Shared speed, 0.15 with the deadband | 79 | 78 | |
| Shared speed, 0.15, no deadband (the proposal) | 0 | 0 | |
| Shared speed, 0.1, no deadband | 0 | 0 | |

So three things are needed together: a contact rule that acts on every tick of an overlap, no deadband, and a setting apart slow enough for six bodies. In the head-on front under Move orders, the hardest scene, a setting apart of 0.3 leaves 7 to 10 shakers of 900 at 1.2 and 1.4; 0.2, 0.15 and 0.1 leave none.

The first idea of this session, the legs fading as a man sinks in, ends the shake but keeps the swap of speeds: the runner still stops in one tick and the man he hits still takes everything. It was set aside for the shared speed. It is in the port as `cushion`.

## The proposal: a contact shares the speed

Not built. It waits for Gota.

1. `steer`, the contact rule, for every man and on every tick an overlap lasts: toward the bodies he overlaps, measured against their own speed (as c1491d1 measures it), he loses his share of the closing speed. His share goes by weight: half against a man of his own weight, less for a heavy man against a light one (`o_mass / (my_mass + o_mass)`, weighted over the bodies by how far each sets him back). The man he runs into applies the same rule from his side and gains what the runner lost. So the two end the tick at one speed along the line between them and go on together. Nothing is thrown away and nothing is handed over whole.
2. `scan` and `yield_to_crowd`, the setting apart: 0.15 of the overlap a tick (0.3 now) on every tick, with no 1 cm deadband.
3. Removed: the beat (`presses` and its four numbers) and the exception for men under a Move order. The brake of e8840e0, the body distance and the spring stay as they are.
4. With `FL_FIGHT_ROOM=0` everything stays as on main, so the fingerprint check still holds.

No new number is tuned. The share comes from the masses the sim already has. The 0.15 is under one sixth, the bound at which six bodies around a man cannot throw him past the middle.

### What the port shows for it

| | Shakers of 1,050 | Hard | Press | Held unit carried by 40 s | Pass | Holder's men off their start at 14 s | Head-on under Move orders, shakers |
|---|---|---|---|---|---|---|---|
| 0.2.1, 0.9 | 250 | 65 | 0.83 m | 3.7 m | 2.33 m/s | 3.0 m | 52% |
| 2548376, 1.0 | 666 | 426 | 0.91 | 5.4 | 1.61 | 2.2 | 60% |
| 2548376, 1.2 | 946 | 860 | | 6.3 | 1.61 | 1.9 | 45% |
| e8840e0, 1.0 | 414 | 161 | 0.91 | 4.0 | 1.58 | 2.3 | 66% |
| e8840e0, 1.2 | 704 | 523 | 1.07 | 4.0 | 1.53 | 1.2 | 75% |
| c1491d1, 1.0 | 97 | 30 | 0.89 | 4.3 | 2.40 | 3.9 | 14% |
| c1491d1, 1.2 | 110 | 46 | 1.03 | 2.9 | 1.96 | 2.4 | 6% |
| 6351066, 1.0 | 128 | 21 | 0.91 | 4.7 | 1.88 | 3.7 | 66% |
| 6351066, 1.2 | 441 | 333 | 1.04 | 4.5 | 1.49 | 3.2 | 75% |
| Proposed, 1.0 | 0 | 0 | 0.91 | 4.4 | 2.59 | 3.7 | 0% |
| Proposed, 1.2 | 0 | 0 | 1.06 | 3.6 | 1.94 | 2.6 | 0% |

More for the proposal:

- No man of the pile turns 10 times in 2 s even when steps down to 0.3 mm are counted (0.0% at 1.0, 0.1% at 1.2, 0.0% at 1.4).
- The press is not frozen. From 30 s to 40 s a man of the pile travels 14 to 22 cm in 2 s and nearly all of it is real travel (path 14.8 cm, net 14.1 cm at 1.2). At e8840e0 the path is 66 cm for 14 cm of travel. 1% of the men move less than 1 cm in 2 s.
- The contact: a front rank running at 9.7 m/s into the back of a standing unit goes to 4.0 m/s on the touch tick and on at about 5 m/s; the men it runs into go from 0 to 5.5 m/s and on at about 3.5. Under the hold the runners go to 0.3 m/s and the men they hit to 9.3 m/s for one tick.
- Mixed weights (archers held, knights and men-at-arms on top): 0 shakers at 1.0, 1.2 and 1.4. The plain pile at 1.1 and 1.4 and the head-on front at 1.1 and 1.4: 0 as well.
- Nearest man in the pile, 1st percentile: 0.80 m at 1.0 and 0.94 m at 1.2, against 0.82 and 0.97 m at e8840e0. Men sink 2 to 3 cm deeper into each other.

### What is removed, and what may feel worse

For Gota's yes on each:

- The beat goes, and with it the idea that a settled press pushes in surges. In the port the press keeps creeping without it.
- Men under a Move order lose their own contact rule. They push through with the shared speed, at 1.9 to 2.6 m/s in the port (main 2.3).
- A man who is run into now moves at a real speed. A unit that holds is pushed about as far as under c1491d1 and 6351066 (port: its men 2.6 m off their start at 1.2, 1.2 m at e8840e0). In the game the measure is the spearwall charge (the wall moved back 1.2 m at e8840e0, 2.3 m under the hold, 1.8 m under the beat).
- Two bodies pushed into each other come apart in about half a second instead of two ticks.
- The 1 cm deadband is gone, so packed men drift by millimetres where they stood still.

### Not known

- Everything with a fight in it: the kill pace of the piles, the direction test, the footwork at the weapon's length, a charge on a spearwall, staggered men. The port has none of it.
- How a press of the proposal looks in motion. The port says it creeps and does not shake; whether that reads as men is for Gota's eye.
- The cost: the scan sums one more number per overlapped body.

## Figures

- `work/notes/vis/038-contact-rules-by-version.png` (script `work/scripts/viz/contact_rules_038.py`): one row per version. A man and his six neighbours to scale at the distance the press settles at, who acts at which distance between two men (the brake, the contact rule, the setting apart, the spring), and the rule in words.
- `work/notes/vis/039-contact-behaviour-by-version.png` (script `work/scripts/viz/contact_behaviour_039.py`, cache in `tmp/runs/soldier-scale/contact-toy/`): one row per version from the port. The speed of a running rank and of the men it runs into around the touch, the path of three packed men over 2 s from above, and the pile from above with the shaking men marked, beside the game's own count.

## Checks for a build of it

The shake log on the six-on-one pile at 1.0, 1.1 and 1.2, on the pass-through and on the 200k scripted front; the push tables of devlog 0208 (ground the attacked front gave, the pace through a friendly unit, the spearwall charge, the two-on-one and wide piles, the direction test); `FL_FIGHT_ROOM=0` against main; a look at screenshots of the press at 200k; then Gota's play at `FL_BODY=1.2`.

## 2026-10-06, later: Gota's answers, and clips of the port

Gota on the proposal: the split by weight is "interesting and worth trying"; the slower setting apart was asked about ("so it'll happen over the course of multiple sim ticks somewhat slowly?"); the removals: "good". Then: "I'd first try this in your python port and see what happens. I feel like m2tw kind of does this too", and whether the port can be watched: "it's cheaper to test on 2D circles than full game". Nothing is approved to build in the game yet.

### The two answers

- The split is the collision of two bodies that do not bounce. Along the line between them both leave at `(m1 v1 + m2 v2) / (m1 + m2)`. With the game's weights (knight 1.5, spearman 1.0, man-at-arms 0.9, bowman 0.8): a knight at 6 m/s into a standing bowman, both go on at 3.9 m/s; a bowman into a knight, 2.1 m/s; equal men, 3.0 m/s. It is not all of real life: it acts only along that line (no rubbing sideways), a standing man's feet hold nothing back (he takes his whole share), the legs act again on the next tick, and with several bodies at once the shares of one tick are summed from the speeds at its start, so a crowd evens out over a few ticks.
- The setting apart at 0.15: each of the two moves 0.15 of the overlap a tick, so 70% of it is left after a tick. Half is gone after 2 ticks, 90% after 7 ticks (0.2 s), 99% after 13 (0.43 s). At 0.3 it is 40% a tick, 90% in under 3 ticks. The "about half a second" told to Gota earlier was the time for nearly all of it. In the port 0.1 leaves no shaker either; over about one sixth the throw past the middle comes back.
- M2TW: not known. Its executable has an impact status per man with a `momentum` field and knockback reactions (devlog 0208), which fits and shows nothing about the rule.

### The weights in the port

`FL_TEST_ROUTPASS` at 1.2, 14 s, how far the holder's men were carried along and the mover's pace:

| | e8840e0 | Proposed |
|---|---|---|
| Knights through bowmen | 1.4 m, 1.19 m/s | 3.1 m, 1.68 m/s |
| Bowmen through knights | 0.6 m, 1.52 m/s | 1.5 m, 1.91 m/s |
| Men-at-arms through men-at-arms | 1.1 m, 1.53 m/s | 2.5 m, 1.94 m/s |
| Knights through knights | 0.9 m, 1.16 m/s | 2.3 m, 1.52 m/s |

A heavy unit carries a light one twice as far as the other way round. Under the proposal a unit that holds is carried about twice as far as at e8840e0 whatever the weights. Shakers at the end: 480 to 567 of 1,020 at e8840e0, none under the proposal.

### The clips

`work/scripts/viz/contact_anim.py <scene>` renders a scene of the port under several versions side by side, seen from above. Every man is a circle one body distance across, one frame is one sim tick, and the clip plays at 30 frames a second, the game's speed. Each panel's head counts the men who shook in the last 2 s. Scenes: `through` (a unit under a Move order through a unit that holds), `knights` and `bowmen` (the same with knights through bowmen and bowmen through knights), `rear` (a unit ordered onto the place of a unit that holds, so into its back), `pile`, `front`. Options: `--close`, `--view x0,y0,x1,y1`, `--versions`, `--body`, `--seconds`, `--mark` (a man is red on a tick he turns against his last step), `--size`. A clip of 25 s takes about a minute.

Rendered at `FL_BODY=1.2` into `tmp/runs/soldier-scale/contact-toy/`, with e8840e0, the hold, the beat and the proposal side by side: `through`, `rear`, `pile` and `front`, each whole and `-close`; `knights` and `bowmen`; `rear-close-body1.2-mark.mp4`; and `through-close-0.2.1-and-proposed.mp4`.

What stills from them show (Claude looked at frames, not at the motion):

- The counts: in `through` at 9 s, 100 of 224 shake at e8840e0, 3 under the hold, 113 under the beat, 0 under the proposal. In `pile` at 30 s: 466, 92, 332 and 0 of 672.
- In `through`, e8840e0 leaves the unit that holds in a neat grid where it stood, and the movers pile up behind it and leak round its flanks. Under the proposal the unit that holds is carried about 4 m by 9 s and squeezed into an arch, a few of its men are left behind where the block stood, and most of the movers go round the flanks (the unit is 14 files wide). The hold scatters the holder's men furthest. The beat is between e8840e0 and the proposal.
- Bowmen sent through knights loosen the knights and flow round them; knights sent through bowmen carry the bowmen's block along.

Open for Gota's eye on the clips: whether a unit that holds is carried too easily under the proposal. A standing man's feet hold nothing against a shared speed, while against the spring they hold the first 6 m/s² (`STAND_GRIP`). Not tried.

## 2026-10-07: built as a6dbe68

Gota watched `front-close-body1.2.mp4`, `through-close-0.2.1-and-proposed.mp4` and others: "the proposed behavior looks good and it also looks like it reproduces the twitch issue of previous builds, esp. the bump one". Then: "If there's no blocker, start implementing."

The change, all in `src/sim/soldier.rs`:

- `steer`: with the room to fight on, one contact rule for every man on every tick of an overlap: toward the bodies he overlaps, measured against their own movement, he loses his share by weight of the closing speed (`overlap_share`, read in `scan` as `o_mass / (my_mass + o_mass)` weighted by how far each body sets him back). The man he runs into applies the same rule from his side and gains the rest.
- `scan` and `yield_to_crowd`: the setting apart at `CORR_GAIN` 0.15 (`corr_gain()`, 0.3 with the room to fight off), and the 1 cm deadband only with the room to fight off.
- Removed: `presses` with `PRESS_TICKS`, `PRESS_BEAT_MIN`, `PRESS_BEAT_MAX` and `PRESS_SKIP`, and the exception for men under a Move order.

Checks at a6dbe68, default body distance unless given (6351066, the hold c1491d1 and e8840e0 in brackets, from devlog 0208):

- Strict clippy clean, 35 tests pass. `FL_FIGHT_ROOM=0` equals main: direction 18 of 18 fingerprints, two-on-one 28 of 28.
- Six-on-one pile, the mean of 20 s to 44 s: 0 of 3,011 shake at 1.0, 0 of 3,051 at 1.1, 0 of 3,101 at 1.2 (6351066: 297, 309, 352; the hold: 79, 19, 31; e8840e0: 1,023 and 1,982 at 1.0 and 1.2). The press 0.87 to 0.92 m (the same). The attacked front gave 7.8 m by 24 s and 7.5 m by 40 s (8.9 and 7.5; 8.0 and 8.1; 6.4 and 6.3), 133 victims left at 40 s (132; 129; 159).
- A unit under a Move order through a friendly unit: 2.24 m/s (2.10; 2.56; 1.73; main 2.29), 0 of 1,000 shake (the hold: 6; e8840e0: 421). At 1.2: 2.09 m/s (e8840e0: 1.39), the holder's disorder 2.5 m at 12 s.
- Spearwall charge at 40 s: wall lane 382 spearmen against 361 knights, the wall moved back 2.1 m; open lane 322 against 403, moved back 1.4 m (6351066: 378 against 362, 1.8 and 1.7 m; the hold: 385 against 355, 2.3 and 1.3 m; e8840e0: 382 against 368, 1.2 and 0.3 m).
- The 200k scripted front: 0 of 200,000 shake at every report to 71 s (6351066: 30,000 to 76,000; the hold: 4,077); 187,179 alive at 71 s (6351066: 186,203 at 75 s).
- The 200k AI battle, one run: 0 shake at every report to 95 s (6351066: 4,400 to 5,900); fighting units 0.95, 0.91 and 0.91 m between comrades at 60, 90 and 100 s (0.96 and 0.91), the tightest 0.86 to 0.87 m; sim step 8.8 to 9.3 ms (9.1 to 9.3), not by the perf rules. No panics. The picture at 105 s, `tmp/runs/soldier-scale/body/ai200k-share_105s.png`: a dense mass in rows with grass between the men, as the pictures before it.

Logs: `tmp/runs/soldier-scale/shake/share-checks.log` (the chain), `share-pile.log`, `share-pass.log`, `share-charge.log`, `share-b11.log`, `share-b12.log`, `front200k-share.log`; `tmp/runs/soldier-scale/pass/share-b12.log`; `tmp/runs/soldier-scale/body/ai200k-share.log` and its two pictures. Fingerprints `tmp/runs/scripts/gates/ss-share-*`.

Open, for Gota's play at `FL_BODY=1.2`: the look of a contact and of the press, a unit forced through another, and the charge on a spearwall, which now moves the wall back 2.1 m instead of 1.2 m at e8840e0 (2.3 m under the hold). If a unit that holds gives way too easily, the standing man's feet against a shared speed are the next thing to try, in the port first.

## 2026-10-07: Gota's play of a6dbe68, and the proposal against the wave

Gota: "I don't see twitching and that's good. I feel this is probably a good physics model as a base layer, but now soldiers do the robotic wave behavior pretty frequently that happened in the hold build and this is uncanny and feels bad for my eyes for some reason and very unpleasant." Screenshot `refs/unit_scale_branch/debug4.png`: the 200k battle, with arrows drawn over waves rolling through the blue press. Asked for: the best fix that keeps the base physics.

### The reading

With the shared speed, every man who arrives at a run at the back of the press hands half his speed to the man he hits, who hands half of that on: a lurch that travels through the rows. The rear units' men keep arriving at 3 to 9 m/s (their drive is `speed * dist / 35` to a slot inside the press, and the brake fades it only once six neighbours are close), so the press is hit all the time, and rows of identical men lurch together. e8840e0 destroyed that speed on the touch tick (the hard stop), so its only waves were the slow setting apart; the hold handed the whole speed on. Two things are missing against real men: nobody runs into the back of a standing comrade (he slows and walks up), and a standing man's feet absorb a gentle shove.

### Tried in the port, `rear` scene (a unit ordered into the back of a unit that holds) at 1.2

Stepping: the share of the holder's men whose smoothed ground speed (the walk signal's 0.25 s) is over 0.3 m/s, the mean of seconds 3 to 8 and the peak.

| | Stepping | Holder moved by 20 s | Pass through a holder: pace, its men off their start | Pile: held unit carried | Shakers |
|---|---|---|---|---|---|
| e8840e0 | 5 to 13% (13%) | 0.81 m | 1.53 m/s, 1.24 m | 3.97 m | 704 |
| a6dbe68, the shared speed | 37% (46%) | 1.76 m | 1.94 m/s, 2.59 m | 3.55 m | 0 |
| Feet planted against a shared speed (`STAND_GRIP`) | 24% (39%) | 1.57 m | 1.99, 2.42 | 3.42 | 0 |
| The same, twice the grip | 27% (37%) | 1.41 m | 2.02, 2.35 | 3.43 | 0 |
| A give: the share grows over the first 15 cm | 25% (38%) | 1.58 m | 1.90, 1.90 | 3.29 | 0 |
| Walk up to a comrade ahead at the advance pace | 22% (40%) | 1.13 m | 1.53, 1.27 | 3.44 | 0 |
| Walk up and feet planted | 15% (23%) | 1.03 m | 1.55, 1.25 | 2.93 | 0 |
| Walk up, feet planted and the give | 15% (20%) | 1.03 m | 1.49, 1.01 | 2.81 | 1 in the front |

The hold, for scale: 49 to 59% stepping, the holder moved 2.05 m. In the port the walk-up rule is: a man under orders, not holding, with a comrade of his team within 2 m, ahead within 45 degrees, who is not walking away along his way faster than 0.5 m/s, has his drive capped at 1.06 m/s (`walkup`, `grip`, `walkup-grip` in `VERSIONS`). Feet planted: a standing man loses a shared speed only above `STAND_GRIP * DT` (0.2 m/s a tick), his own grip 0.7 to 1.3 of it.

Clips in `tmp/runs/soldier-scale/contact-toy/`, e8840e0, a6dbe68, walk up, and walk up with feet planted side by side: `rear-walkup.mp4`, `rear-close-walkup.mp4`, `through-walkup.mp4`, `pile-close-walkup.mp4`.

### The proposal, not built

1. Walk up to a comrade ahead. In `look_around`, a man under orders walking to his slot looks along his way as a fighter looks along his way to an enemy (the `ahead` test already there: a comrade within the scan's reach, in front, not walking away). With one ahead his drive is capped at `ADVANCE_PACE`, 1.06 m/s, the pace a fighter already takes with a comrade close ahead. He keeps walking and pushing; he does not wait.
2. Feet planted against a shove. In `steer`, a standing man (the spring's test, `desired` zero) takes a shared speed only above `STAND_GRIP * dt`, his own grip 0.7 to 1.3 of it.

Removed: nothing. The base physics stays. What may feel worse: a unit forced through a friendly unit walks through at the pace of e8840e0 again (port: 1.5 m/s instead of 1.9 at 1.2); men joining a press walk the last 2 m; a unit that holds yields a little less. The look pass runs every 8th tick for the marching men too, a few percent of the scan at 200k.

## 2026-10-07, later: the walk-up proposal rejected, and the map of the contact

Gota: "This sounds very superficial and treating symptoms not fundamental problem. Why would soldiers in a battle suddenly walk up when there's a chance to close the gaps getting close to the frontline or their destination? ... I didn't feel this wave issue at all in 0.2.1 build, and soldiers looked like they bump into slightly different directions or something, making it organic. Propose a fundamental fix while keeping the good aspects of the build." The walk-up rule was wrong: it lowered the arrival speed by a rule about intent instead of fixing what the contact does with the speed.

### Where the energy of an arrival goes

Every man who arrives at the back of a press at 3 to 10 m/s (his drive is `speed * dist / 35` to a slot inside it) brings kinetic energy, and the contact has to put it somewhere. The port's `rear` scene (a unit ordered into the back of a unit that holds, at 1.2) measures the result as the share of the holder's men stepping (smoothed ground speed over 0.3 m/s, seconds 3 to 8, and the peak), how far the holder is carried by 20 s, the pace of a Move-ordered unit through a holder with how far its men are pushed off, how far the pile's held unit is carried by 40 s, and the shakers in the four scenes.

| Where the speed goes | Stepping | Holder carried | Pass, holder off | Pile carried | Shakers rear / pass / pile / front |
|---|---|---|---|---|---|
| Into the man he hits, half (a6dbe68) | 37% (46%) | 1.76 m | 1.94 m/s, 2.59 m | 3.55 m | 0 / 0 / 0 / 0 |
| The same, feet planted (`STAND_GRIP` on the share) | 24% (39%) | 1.57 | 1.99, 2.42 | 3.42 | 0 / 0 / 0 / 0 |
| The same, the man hit weighs as much as the men he leans on | 37% (45%) | 1.70 | 2.03, 2.44 | 3.83 | 0 / 1 / 0 / 0 |
| Nowhere at once: a hard stop with the 1 cm deadband (e8840e0) | 8% (13%) | 0.81 | 1.53, 1.24 | 3.97 | 61 / 554 / 704 / 672 |
| Into a lean: a soft wall, give 15 cm, nothing handed on | 22% (30%) | 1.56 | 1.81, 1.47 | 3.21 | 0 / 0 / 0 / 0 |
| Soft wall, a standing man's stance holds the first 10 cm of a squeeze | 19% (26%) | 1.18 | 1.55, 1.25 | 2.24 | 0 / 0 / 0 / 0 |
| Soft wall, give 8 cm, stance 10 cm | 14% (22%) | 1.06 | 1.73, 1.23 | 2.46 | 0 / 0 / 0 / 7 |
| Soft wall, give 5 cm, stance 10 cm | 13% (20%) | 0.91 | 1.55, 1.42 | 2.49 | 0 / 1 / 12 / 71 |
| Soft wall, give 2 cm, stance 10 cm | 9% (12%) | 0.70 | 1.82, 1.23 | 3.50 | 17 / 5 / 30 / 122 |
| Not brought: he brakes for a comrade ahead as his legs can (7 m/s²) | 9% (23%) | 0.56 | 0.52, 0.71 | 1.59, no press | 0 / 0 / 0 / 0 |
| Brakes and the soft wall | 1% (4%) | 0.20 | 0.77, 0.32 | 1.16, no press | 0 / 0 / 0 / 1 |
| Soft wall and stance, the spring held flat inside the body | 13% (21%) | 1.06 | 2.45, 3.58 | 6.08, press 0.70 m | 0 / 5 / 0 / 0 |

Read:

- Handing the speed on (the share) makes the wave; feet planted take a third off it; weighing a man by the men he leans on does nothing in a unit at slot spacing, where nobody overlaps anybody.
- Destroying the speed on the touch tick is what 0.2.1 and e8840e0 did, and it is the twitch: with a give of 5 cm or less the shakers come back (12 to 122).
- A soft wall with a give of 8 to 15 cm and a stance (the physical replacement of the old deadband: a standing man gives ground only beyond the first 10 cm of a net squeeze) is the nearest to e8840e0 without the twitch: 14 to 19% stepping, the holder carried 1.1 to 1.2 m, the pass at e8840e0's pace. The residual is the lean itself: an arriving man sinks his give, and from there the spring (its push inside the body distance) moves the standing man.
- Braking for a comrade removes the wave and the press with it: men stop short of each other and never pack. Holding the spring flat inside the body distance collapses the press to 0.70 m.
- `walk-up` (the rejected proposal) is kept in the port as `walkup`.

Clips in `tmp/runs/soldier-scale/contact-toy/` with e8840e0, a6dbe68, the soft wall with give 15 cm and stance, and with give 8 cm: `rear-close-wall.mp4`, `rear-wall.mp4`, `through-wall.mp4`, `pile-close-wall.mp4`.

## 2026-10-07: the soft wall built as the next commit after a6dbe68

Gota: "Ok fine let's try it then. As long as I've watched those vids, give 8cm parameter looks more interesting."

The change in `src/sim/soldier.rs`, replacing the shared speed:

- `steer`: toward the bodies he overlaps, a man's legs lose their closing speed over his give (`GIVE` 0.08 m, each man's own 0.7 to 1.3 of it by `give(i)`): all of it is left at the touch, none at the give, never carrying him past it in one tick. The closing speed is measured against the bodies' own movement and never more of it is taken than his own speed into them. Nothing is handed to the man he hits.
- `steer`, the stance: a standing man (the spring grip's test, `desired` zero; not a staggered man) gives ground to a net squeeze only beyond its first `STANCE` 0.10 m: `st.corr` scaled by `(depth - STANCE) / depth`, where `overlap_depth` is the net overlap along the correction before its cap.
- Removed: `overlap_share`. Kept: the setting apart at 0.15 on every tick, the brake, the spring, the mass weights.

Checks (a6dbe68 and e8840e0 in brackets):

- Strict clippy clean, 35 tests pass. `FL_FIGHT_ROOM=0` equals main: direction 18 of 18, two-on-one 28 of 28.
- Six-on-one pile, 20 s to 44 s: 0 of 3,046 shake at 1.0; 4 of 3,073 (2 hard, step 0.4 cm) at 1.1; 4 of 3,104 (2 hard) at 1.2 (a6dbe68: 0, 0, 0; e8840e0: 1,023 and 1,982). The press 0.87 to 0.91 m. The attacked front gave 6.5 m by 24 s and 5.1 m by 40 s (7.8 and 7.5; 6.4 and 6.3), 156 victims left at 40 s (133; 159): the push on an enemy pile is back at e8840e0's level.
- A unit under a Move order through a friendly unit: 2.19 m/s (2.24; 1.73), 0 of 1,000 shake. At 1.2: 1.78 m/s (2.09; 1.39), the holder's disorder 1.84 m at 12 s (2.53).
- Spearwall charge at 40 s: wall lane 374 spearmen against 364 knights, the wall moved back 1.3 m; open lane 310 against 406, 0.6 m (a6dbe68: 382 against 361, 2.1 and 1.4 m; e8840e0: 382 against 368, 1.2 and 0.3 m). The wall holds as at e8840e0 and loses 8 more spearmen.
- The 200k scripted front: 0 of 200,000 shake at every report to 71 s; 187,170 alive at 71 s (187,179).
- The 200k AI battle, one run: 0 shake from 30 s on (2 at 14 s); fighting units 0.87 and 0.88 m between comrades at 60 and 100 s (0.95 and 0.91): the press sits about 5 cm tighter, the stance letting a squeeze stand; sim step 9.3 ms (8.8 to 9.3); 183,057 alive at 95 s (187,292; the battle is not deterministic). No panics. Picture at 105 s, `tmp/runs/soldier-scale/body/ai200k-wall_105s.png`: the dense mass in rows as before.

Logs: `tmp/runs/soldier-scale/shake/wall-checks.log` and `wall-*.log`, `front200k-wall.log`; `tmp/runs/soldier-scale/pass/wall-b12.log`; `tmp/runs/soldier-scale/body/ai200k-wall.log`. Fingerprints `tmp/runs/scripts/gates/ss-wall-*`.

The port: `proposed` in `contact_toy.py` is now this rule (give 8 cm, stance 10 cm) and `a6dbe68` the shared speed. Figures 038 and 039 still draw the shared speed as the proposal; they are not redrawn.

For Gota's eye at `FL_BODY=1.2`: the arrival at a press (the bump), whether the rows still lurch, the press (5 cm tighter at 200k), a unit forced through another, the charge.

## 2026-10-07, later: Gota's play of 2f81fe6, and the gap rush

Gota: "initial clash wave: it's subtle and ok level but could be better toward the 0.2.1 feel. At least it doesn't do stupid robotic wave like the hold build. But debug4.png like waves still happen and I don't like it. It's still not fixed. It's probably like 0.2.1 had the same issue, but soldier spacing was much wider so it didn't reach to a point where I notice it as an issue." Then, on Claude's first reading (a two-circle body): "You're so clueless. It happens when soldiers in frontline dies or whatever and a gap emerges and everyone closing the gap by running into it and a wave occurs. Test in python first."

### The gap rush in the port

The pile at 1.2 with deaths at the contact from 12 s on, three a second (`Sim.kill`, `Sim.contact_men`). The wave measure: a man's smoothed velocity minus the mean over 8 m around him, then how much of that his neighbours within 2.5 m share; "gap rush" is the share of packed attackers whose neighbours move together at over 8 cm/s, from 14 s to 45 s.

| | Gap rush | The press |
|---|---|---|
| 0.2.1 at 0.9 | 2% | 0.85 m |
| e8840e0 at 1.2 | 5% | |
| a6dbe68 at 1.2 | 9% | |
| 2f81fe6 at 1.2 | 14% | 1.05 m |
| 2f81fe6 at 1.1 | 13% | 0.97 m |
| 2f81fe6 at 1.0 | 6% | 0.90 m |
| 2f81fe6 at 0.9 | 3% | |

What does not touch it at 1.2: a cap on the legs' acceleration (7 or 4 m/s²: 15 to 16%), reaction windows before a braked man's drive returns (0.5 or 1 s: 15%), the setting apart at 0.05 (15%), a stance for every man (13 to 16%), two circles with a hard core at 1.0 or 0.9 and a soft zone to 1.2 (13 to 16%, whatever the soft zone's rate). The rush is not the legs' acceleration, not the men's timing, not the squeeze relaxing.

What does: the brake's band. At 1.2 the e8840e0 band runs from six neighbours at the touch (1.2 m) to six at 1.09 m, 11 cm of squeeze, so one death beside a man hands him back half his drive at once. With the band's end at six at 1.0 m: 6%; at 0.9 m: 5%; with the band starting inside the body (six at 1.12 m, the 0.9 and 1.0 start): 21 to 29%. But the same widening at 1.0 (end at 0.8 m) makes it worse (10 to 12%, the press 0.79 to 0.83 m), and a band from the touch to 20 cm in is wrong at 0.9 and 1.0 (35 to 46%, the pass at 4 m/s). So the band is not a width in metres of the body distance.

One band for every body distance, anchored on the man as drawn: his drive fades from six neighbours at 1.2 m (his arms and shield meeting theirs, the drawn width plus 0.2 m) to six at 1.0 m (the drawn bodies touching). The sim's body circle (`FL_BODY`) is then only how far apart men are set; what a man feels as crowded is the other men's bodies.

| `FL_BODY` | Gap rush | The press | Pass through a holder | Shakers pile / pass / front | Rear stepping, holder moved |
|---|---|---|---|---|---|
| 0.9 | 1% | 0.89 m | 1.93 m/s | 0 / 0 / 0 | 23%, 1.25 m |
| 1.0 | 2% (now 6%) | 0.95 m (0.90) | 1.79 m/s (2.10) | 0 / 0 / 0 | 23%, 1.26 m |
| 1.1 | 6% (13%) | 0.99 m (0.97) | 1.64 m/s (2.05) | 0 / 0 / 0 | 23%, 1.16 m |
| 1.2 | 6% (14%) | 0.97 m (1.05) | 1.91 m/s (1.73) | 0 / 0 / 4 | 19%, 1.15 m |

The arrival wave (rear stepping) does not change: it is the other thing, and Gota called it subtle and ok. The press at 1.2 sits 8 cm tighter in the port. In the port this is `drawn-brake` in `VERSIONS`. Clips with deaths (`--deaths` in `contact_anim.py`), 0.2.1 at 0.9, e8840e0, 2f81fe6 and the drawn-width brake side by side: `tmp/runs/soldier-scale/contact-toy/pile-close-deaths.mp4`, `pile-deaths.mp4`.

Also tried and set aside on the way: braking for a comrade ahead (removes the wave and the press with it), walking up to a comrade (rejected by Gota), the spring held flat inside the body (the press collapses to 0.70 m).

### The two-and-four scene (Gota's scene)

Gota: "pile-deaths.mp4 is a bad scenario that doesn't reproduce this issue at all and whatever you measured is meaningless. Try putting two blue units of reasonable column width (a bit wider than the previous one) with some gap between them and have four red units attack them." Built as `two_and_four` in the port (`twofour` in the clip tool): two units of 16 by 8 hold 6 m apart, four units of 16 by 8 attack them, two on each, one behind the other; deaths at the contact every 8 ticks from 12 s. Clips: `tmp/runs/soldier-scale/contact-toy/twofour-deaths.mp4`, `twofour-close-deaths.mp4` (0.2.1 at 0.9, e8840e0, 2f81fe6, the drawn-width brake).

Measured death by death: of the dead man's own side within 4 m, how many move toward his place in the next 1.5 s.

| | Men within 4 m | Move over 10 cm toward it | Over 30 cm |
|---|---|---|---|
| 0.2.1 at 0.9 | 19.9 | 7.5 | 2.3 |
| e8840e0 at 1.2 | 19.0 | 7.4 | 2.2 |
| 2f81fe6 at 1.2 | 20.5 | 7.9 | 2.2 |
| 2f81fe6 at 1.0 | 21.0 | 8.2 | 2.5 |
| The drawn-width brake at 1.2 | 19.7 | 7.6 | 1.7 |

In the pile with deaths the same: 11 to 14 men move over 10 cm per death in every version, 0.2.1 included. So in the port a death closes the same way at 0.9 and at 1.2, and the drawn-width brake changes nothing here: that proposal is withdrawn. The port cannot show the difference Gota sees. Either it is in what is seen (men drawn 0.59 m wide at 0.9 leave air around a 20 cm move; men 1.01 m wide at 1.05 m spacing make the same move a wave of touching bodies), or it is in what the port lacks: the fighters' footwork (a death frees an enemy, and the men behind advance on him at 1.06 or 2.87 m/s), men joining a fight, a unit re-dressing after a death. The next measurement has to be in the game.

### The push scene (2026-10-07, late)

Gota: the two-and-four scene and the pile with deaths are garbage and were deleted (code and clips); "gap between two blue needs to be tighter, introduce the third blue and the red mass should try to move past blue ranks, dont put the blue at the top edge of the map". The port also had no fight, so a mass just flowed through a line; it now has the least of one (`Sim.fight`: a man with an enemy within `REACH` 2 m goes at him at the advance pace and stands at `FIGHT_AT` 1.8 m). `masses()` builds fronts; the scene `push`: three blue units of 16 by 8 hold 2 m apart in the middle of the field, a red mass of twelve units (four across, three deep) is Move-ordered to a point 60 m behind them, deaths at the contact every 4 ticks from 10 s. Clips: `push-deaths.mp4`, `push-close-deaths.mp4`, and `-drawn` versions with every man drawn at his drawn width (0.59 m in 0.2.1, 1.01 m now) instead of the body distance. The scenes `wall`, `line`, `clash`, `deep` from the same builder are there too.

Measured on the red mass from 14 s to 45 s: "wave patches", the share of packed men whose neighbours within 2.5 m share a residual motion over 8 cm/s after the mean over 8 m is taken out, and its mean; the share of men stepping (over 0.3 m/s); per death, how many of the dead man's side within 4 m move 10 and 30 cm toward his place in 1.5 s.

| | Wave patches | Stepping | Per death, 10 / 30 cm | Shakers | The blue line pushed |
|---|---|---|---|---|---|
| 0.2.1 at 0.9 | 30% (7.3 cm/s) | 30% | 17.4 / 9.7 | 33 | 6.2 m |
| e8840e0 at 1.2 | 20% (5.4) | 35% | 14.5 / 8.8 | 953 | 6.3 m |
| a6dbe68 at 1.2 | 45% (14.2) | 41% | 10.6 / 7.2 | 2 | 6.8 m |
| c1491d1 at 1.2 | 51% (15.5) | 42% | 12.9 / 8.2 | 61 | 7.7 m |
| 2f81fe6 at 1.0 | 65% (18.2) | 41% | 14.2 / 7.5 | 2 | 4.2 m |
| 2f81fe6 at 1.2 | 44% (9.6) | 27% | 11.9 / 6.9 | 0 | 1.9 m |

The wave-patch measure puts the hold and the shared speed above 0.2.1, as Gota's eye did, and 2f81fe6 at 1.2 between them; e8840e0 is lowest because its motion is the twitch (953 shakers). At 1.0 the red mass bursts through the 2 m gaps and the measure reads the flow. The per-death count does not follow Gota's eye (0.2.1 closes a gap with more men than 2f81fe6).

Candidates at 1.2 on this scene (wave patches, mean): stance for everyone 43% (10.6); setting apart 0.08: 51% (12.6); give 15 cm 44% (12.0); the spring at half 32% (7.8); the drawn-width brake 39% (9.1), the blue line pushed 3.3 m; two circles 46% (11.4); legs capped at 7 m/s² 37% (8.2); a stance of 20 cm 46% (10.4). Two move it toward 0.2.1: the spring at half strength and a human cap on the legs. Not yet understood why, and not checked for what else they change.

### The push scene reproduces it (Gota), and what cures it in the port

Gota on `push-deaths-drawn.mp4`: it reproduces the issue; under the shared speed "from around 00:25 multiple waves as the red soldiers try go through the gaps opened between the right most blue and middle one"; the soft wall's clip ended before its gaps opened. Run for 100 s (`push-deaths-drawn-100s.mp4`, `push-close-drawn-100s.mp4`, the close view on the right gap), deaths five a second. Two windows measured on the red mass: the flank flow round the line's end (20 to 55 s, within 15 m of x = -38) and the breach (within 15 m of where the first five red men stand behind the line between its ends, for 35 s from then). Wave patches as before.

| At 1.2 unless given | Flank flow | Breach, 35 s on | Shakers |
|---|---|---|---|
| 0.2.1 at 0.9 | 9% (4.1 cm/s) | 20% (6.2 cm/s) | 0 |
| 2f81fe6 | 22% (6.4) | 45% (9.4) | 1 |
| a6dbe68 | 16% (5.5) | 41% (10.6) | 0 |
| The 0.2.1 rule at 1.2 (2548376) | 23% (5.4) | 43% (8.9) | 385 |
| Brake 1.2 to 1.0 m (drawn width) | 23% | 42% (9.4) | 1 |
| Brake 1.12 to 0.82 m (0.2.1's) | 14% | 34% (9.2) | 1 |
| Brake from the touch to 0.9 m | 19% | 38% (8.7) | 3 |
| Brake 1.3 to 0.9 m | 23% | 39% (9.4) | 5 |
| Legs capped at 4 m/s² | 13% (3.9) | 29% (6.6) | 0 |
| Legs 4 m/s² and the brake 1.3 to 0.9 m | 12% (3.7) | 17% (4.7) | 0 |

The 0.2.1 rule at a 1.2 body waves as much as any: it is the body distance. Two things together bring 1.2 to 0.2.1's level: a human limit on the legs' acceleration (the steering `(desired - v) * STEER_GAIN`, capped at 4 m/s²; the spring push and the contact are not legs), and the brake's band in metres of real crowding, six neighbours at 1.3 m to six at 0.9 m, whatever `FL_BODY`. The port's leg cap is `Sim.leg_accel`, the band `slow`/`stop`.

What else the pair changes (the port): at 1.2 the press 0.98 m (1.05), the pile's held unit carried 4.85 m (2.46), the pass 1.80 m/s (1.73), the holder's men 1.13 m off (1.23), the arrival wave in `rear` 7% stepping (14%), no shakers anywhere. At 1.0: press 0.94 (0.90), carried 2.55 (4.04), pass 1.76 (2.10), rear 13% (26%).

### Seeing the waves: the space-time picture of the gap (figure 040)

Gota asked whether the proposal came from observing the waves or from guessing. From guessing: Claude cannot watch a clip, had looked at single frames and an aggregate number, and the breach window for the soft wall (18 to 53 s about x = 27) missed the one wave Gota saw at 55 s in the right gap. `work/notes/vis/040-gap-waves-space-time.png` (script `work/scripts/viz/gap_waves_space_time_040.py`): for the red men in the right gap's column (x 6 to 18 m), the mean speed toward the blue line in 1 m bands of y, tick by tick over 100 s, for 0.2.1 at 0.9, a6dbe68, 2f81fe6 and the candidate (legs 4 m/s², brake 1.3 to 0.9 m).

What it shows: under the shared speed (8 to 30 s) and the soft wall (8 to 45 s, and again before the breakthrough at 55 s) the picture has vertical stripes: the whole column of men behind the gap, 20 m deep, speeds up and slows down together, every 1.5 to 2 s, between about 10 and 35 cm/s. That is the wave: the lane behind a gap moving stop and go as one. 0.2.1 flows through steadily (over 40 cm/s, no stripes). The candidate flows steadily at 10 to 25 cm/s with no stripes.

## 2026-10-07: the leg cap and the brake by the men's size, built as the commit after 2f81fe6

Gota agreed to both parts, the second as a first commit to be made flexible: the brake's two distances follow the unit scale (`BRAKE_FAR` 1.3 m and `BRAKE_NEAR` 0.9 m at the 1.70 m man, times `FL_UNIT_SCALE`), so a player's unit scaling moves them and `FL_BODY` stays the separate spacing. The leg cap is `LEG_ACCEL` 4 m/s² (`FL_LEG_ACCEL`), on the steering term only. Both with the room to fight on; `FL_FIGHT_ROOM=0` keeps main's brake and no cap.

Checks (2f81fe6 in brackets):

- Strict clippy clean, 35 tests pass. `FL_FIGHT_ROOM=0` equals main: direction 18 of 18, two-on-one 28 of 28.
- Six-on-one pile: 0 shakers at 1.0, 1.1 and 1.2 (0, 4, 4). The press at 1.0 0.92 to 1.05 m (0.87 to 0.91). The attacked front gave 3.6 m by 24 s and 2.6 m by 40 s (6.5 and 5.1), 211 victims left at 40 s (156): the pile kills a third slower.
- Through a friendly unit: 1.76 m/s at 1.0 (2.19), 1.94 m/s at 1.2 (1.78), 0 shakers.
- Spearwall charge at 40 s: 393 spearmen against 381 knights, the wall moved back 0.8 m (374 against 364, 1.3 m): fewer deaths on both sides.
- The 200k scripted front: 0 shake to 71 s; 188,232 alive at 71 s (187,170).
- The 200k AI battle, one run: 0 shake throughout; fighting units 0.98 and 1.04 m between comrades at 60 and 100 s (0.87 and 0.88); sim step 8.7 ms; 192,682 alive at 95 s (183,057): the battle kills about half as fast. The picture at 105 s, `tmp/runs/soldier-scale/body/ai200k-legs_105s.png`: rows with grass between the men across the whole mass, an open press.
- With the cap off (`FL_LEG_ACCEL=100`), the band alone: the pile front gave 4.0 and 4.5 m, 193 left at 40 s; the pass 1.49 m/s; the charge 394 against 378, 1.0 m. So the slower killing is mostly the band's early start at arm's length (men ease off their drive sooner, the press is looser and fewer men reach an enemy), and the leg cap adds a little to it while making the pass faster.

Logs: `tmp/runs/soldier-scale/shake/legs-checks.log`, `legs-*.log`, `legs-off-*.log`, `front200k-legs.log`; `tmp/runs/soldier-scale/body/ai200k-legs.log` and pictures.

For Gota's play at 1.2: the lane behind a gap, the arrival, the open press, and the slower killing, which is a balance to set apart from the physics (the brake's start at arm's length or the combat scale).

### The push scene in the game (the commit after 8bf1152)

Gota asked for the port's scene in the game, then shaped it: the line much deeper, more men in it so the mass clogs, real gaps. `FL_TEST_PUSH=1` (`spawn_push_test` in `src/regiments.rs`): three orange regiments of 384 men in 17 files and 23 ranks hold in a line at the centre of the field with 3 m of air between their outer men, set on their slots at spawn (a regiment of 384 is laid 30 wide by default and would still be re-dressing when the mass arrives); a blue mass of twelve regiments of 128, four across and three deep, 30 m in front, Move-ordered to a point 60 m behind the line. The line never breaks: `steadfast` on `GroupData`, which `update_morale` honours, because without it the three regiments broke at 16 s and the battle ended in a victory screen. Gota's wedge formation for the line is not possible yet: the game has `Rect` and `Blob` only. With the game's own fight the line loses about 13 men a second (1,152 to 416 by 56 s, the mass 1,536 to 1,351, all of it past the line by then); `FL_COMBAT_SCALE` slows that. Gota: the last run is good enough. Screenshots of it: `tmp/runs/soldier-scale/body/push-legs_{2,30,60,90}s.png`; at 30 s the mass pours through both gaps and round the ends in thick lanes.

## 2026-10-07: Gota's play of the leg cap build, and the state at the end of this session

The comparison switches went in after the push scene (9518d48): `FL_BRAKE=body` plays the brake from the soft-wall build, `FL_CONTACT=share` the shared speed, `FL_LEG_ACCEL=100` takes the leg cap off. In the push scene at `FL_BODY=1.2`, `FL_LEG_ACCEL=100 FL_BRAKE=body` (the soft-wall build as played the night before) reproduces the waves; the current build does not.

Gota, after 5 to 10 minutes over several scenes: "it feels alright. Subtle small waves still happen at the rare condition of stacked units trying to move past the gap between a unit and map edge, but even when this happens it's better than the previous ones ... I'd need to play test more, but I'd say the wave issue is fixed. The rest is concluding how the soft wall logic feels but I don't feel any problems, but I'm not confident yet if it's ready for merge."

Two ideas of Gota's to keep: the port's 2D circles could sell as a lightweight online multiplayer game, like War of Dots; and testing formation and crowd behaviour in a toy Python port before the game is a method to remember for any complex rule.

The state: `feat/soldier-scale` at 9518d48, 26 commits on main 66044d1, not pushed, no PR. This session's commits: a6dbe68 (the shared speed), 2f81fe6 (the soft wall and the stance), 8bf1152 (the leg cap and the brake by the men's size), a2d8efc and b3b0157 (the push scene, `FL_TEST_PUSH`), 9518d48 (the comparison switches). Open: more play of the soft wall before a PR; the slower killing (the pile a third slower, the 200k battle about half), a balance to set; the residual waves at a map edge; the body distance value (below); figures 038 and 039 still show the shared speed as "proposed"; the open items of handoffs 096 and 098 (rule 2, archers at true size, the `FL_` sort, the PR text).

### The body distance that matches 0.2.1

Gota: "How to make FL_BODY equal to the equivalent gap between soldiers in 0.2.1?" From devlog 0208 and handoff 096, two readings of "equal", both from the drawn widths (0.2.1: knights 0.71 m, men-at-arms 0.59 m at the 0.9 m body; now 1.09 and 1.01 m):

- The same air between two men's drawn bodies: 0.2.1 had 0.19 m (knights) to 0.31 m (men-at-arms). At true size that is `FL_BODY` 1.29 m for knights and 1.32 m for men-at-arms: about 1.3.
- The same proportion of body distance to drawn width: 1.39 m for knights, 1.53 m for men-at-arms. 1.53 is over the slot pitch, and `FL_BODY` is capped at the pitch (`SEP_RADIUS`, 1.4), so this reading needs the slots widened too (`FL_GAP`), and the cap in `body_distance()` raised with them.

One `FL_BODY` serves every kind, and the kinds' widths differ (bowmen 0.75 m), so neither reading is exact for all; a body distance per kind would be. Since 8bf1152 the brake no longer moves with `FL_BODY`, so changing it changes only how far apart men are set.

### The body distance, M2TW on the bridge, and the widening (Gota, 2026-10-07 evening)

Gota: `FL_BODY` 1.0 to 1.3 all look good (1.3 the 0.2.1 feel, 1.0 packed but realistic); it should be a settings item, with the question of the default open. Screenshots of M2TW on the river map in `refs/unit_scale_branch/m2tw/` (1 to 6): a mass pushing over a narrow bridge packs until shields overlap and heads touch (3, 4), the units on the far bank are pushed back by it (5), and the line is bent into a V by the end (6); before the push the same units stand in open ranks (1). Read against the engine's 0.8 m collision distance and its soft bodies (devlog 0208: a third of marching men under 0.8 m, the first percentile 0.51 m), the packed push is men at about 0.6 to 0.8 m between centres for bodies 0.8 m across; at flanks' 1.0 m wide men that is `FL_BODY` about 0.9 under pressure, 0.8 with the overlap of shields; a unit fighting in the open sits loose, 1.4 to 1.5 m (devlog 0177). One body distance cannot be both; Gota's two circles (a hard one at the body, a soft one at the combat distance, recovered over seconds, ignored under a Move order) would. Not built. What is on the branch for a fighting man's room: rule 1 (he fights at 0.9 of his reach and steps back to it) and rule 2 (the lane); devlog 0182 measured that they thin the contact and do not open a mass. No rule widens a packed unit when its order ends or turns into an attack; the setting apart takes it to `FL_BODY` at once and the slots re-dress it where men can reach their marks.

Deferred by Gota to another branch: a unit being pushed only once it is down to its last four ranks.

### The end of the thread (2026-10-07 evening)

Gota: M2TW's armies are small (under 20k, often under 10k), so what shows at 200k would not show there. Settings for spacing, pitch and size go on another branch; the default body distance is 1.2 (`BODY_DISTANCE` in the code is still 1.0: a one-line change for the next thread). The widening (Gota's two circles) is the next thread, and the branch wraps up after it is understood and settled. Handoff: `work/handoffs/099-soldier-scale-waves-and-the-soft-wall-2026-10-07.md`.
