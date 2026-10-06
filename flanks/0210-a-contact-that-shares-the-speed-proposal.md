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
