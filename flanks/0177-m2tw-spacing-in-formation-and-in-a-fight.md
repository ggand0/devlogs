# 0177: M2TW spacing, standing in formation and in a fight

Written by Claude Fable 5.1.

2026-10-03. Follows devlog 0176 (the soldier scale try-out). Gota played M2TW again to see why 0.2.1's scale and gaps felt better than the 1.64 try-out, and took 26 screenshots. This devlog keeps Gota's observations as written, what the screenshots show, what M2TW's data files say, and where flanks sits against them. No game code changed. Gota's direction: research M2TW's behaviour first, before the scale branch.

## Gota's observations, as typed

```text
I played m2tw again and I now understand why the current scaling and gaps between soldiers felt better compared to the 1.64 scaling.

1. I played to see the gaps between soldiers in formation (non-engaged) but I think the current spacing is too tight and it should be the gap in shield wall / spear wall
I took screenshots of units here, even pikemen spearwall have gaps wider than the 1.64 scaling. see several screenshots here (down size them): /home/gota/ggando/gamedev/flanks/refs/unit_scale_1.64/m2tw_spacing/formation

2. Next, I checked the spacing when units are engaging, and it's much wider than I thought. I think this is why I kind of liked the previous spacing in battle (for the formation I thought it was a bit wide, like I said at the beginning). Named images here def worth observing: /home/gota/ggando/gamedev/flanks/refs/unit_scale_1.64/m2tw_spacing/spacing_attack-vs-move

When a unit is engaging in attack order or idle and naturally attracted to the enemy, the spacing is kind of similar to the gaps in the 0.2.1 situation. I then did a move order into the middle of enemy unit, and realized that now soldiers going through gaps in a tight way. So I think the spacing parameter of m2tw changes based on situation. Next I did the same with spearman, and then switched to the attack order and I saw the spacing of soldiers in this unit gradually spread and spearmen maybe kind of got pushed each other a little bit? I saw soldiers in outer rim doing shuffling animation to make space (moving away from the center of unit a bit gradually). spearman_move-order.png -> spearman_then-attack-order_spacing-widened.png

worth documenting these observations in a designated devlog and
We should research m2tw behavior first
```

## The screenshots

All in `refs/unit_scale_1.64/m2tw_spacing/`, 2560x1440. Shrunk copies and contact sheets for reading: `tmp/runs/scale/m2tw/`.

`formation/` (units standing or marching, not engaged; the green discs are M2TW's selection markers, one under each man):

| File | What it shows |
|---|---|
| `eng_early_knight.png` | English dismounted knights in close order, seen along the rank. A body's width of open ground between neighbours, discs just apart |
| `eng_early_spearman.png` | Spearmen from the front. Shields nearly edge to edge, discs touching or overlapping |
| `eng_early_archers.png` | Archers in close order, the same pitch as the others |
| `eng_early_loose_formation_overhead.png` | From above: one unit in loose order beside units in close order. The loose gaps are about twice the close ones |
| `eng_early_diagonal_look.png` | The army from above at an angle |
| `fr_pike_marching.png` | French pikemen marching, pikes raised |
| `hre_forlorn_hope.png`, `hre_late_knight.png`, `hre_late_crossbow.png`, `hre_late_landsknecht.png` | Late Holy Roman Empire units in close order; the Landsknecht pikes levelled |
| `Screenshot from 2026-10-03 15-28-25.png` | Pikemen with levelled pikes receiving an attack |
| `Screenshot from 2026-10-03 15-29-22.png` | Zwei Hander standing in close order after a fight |

`spacing_attack-vs-move/`:

| File | What it shows |
|---|---|
| `knight_attack-order_spacing.png` | Knights under an attack order. At the contact the fighters stand well apart, each pair with room; the men behind who have not reached the fight are still packed |
| `knight_move-order_spacing.png` | The same knights under a move order into the enemy: packed tight, passing through gaps |
| `spearman_move-order.png` | Spearmen moved into the enemy: a tight clump, discs overlapping |
| `spearman_then-attack-order_spacing-widened.png` | The same spearmen a little after the attack order: the unit has opened up, discs apart, men on the rim further out |
| ten `Screenshot from 2026-10-03 15-46-32` to `15-49-15` | The run of the test: deployment, the charge, the fight from several heights. From above (15-47-38) the fighting part of each unit is visibly looser than the parts still marching |

## What M2TW's data says

Unpacked today from Gota's install with the game's own unpacker (the steps in memory `m2tw-data-extraction`); the files are in this session's scratchpad and are not kept.

- `export_descr_unit.txt`: every one of the 222 infantry units has `formation 1.2, 1.2, 2.4, 2.4`. That is 1.2 m side to side and front to back in close order, 2.4 m in loose order. Archers, crossbowmen, knights, spearmen and pikemen all carry the same four numbers. What differs per unit is the default rank count (2 to 8) and the special formations it may form (schiltrom for 18 units, phalanx for 19). Cavalry is 2 by 4.4 m close, 3 by 6 loose.
- The `soldier` line has four fields in every vanilla unit: model, count, extras, collision mass. No vanilla unit sets a collision radius, so the radius modders describe (devlog 0036) is an engine default here.
- `descr_skeleton.txt`: each weapon skeleton has a `strike_distances` line of five numbers, and its attack clips carry a distance letter a to e and the point where the weapon lands (`-id`). The file does not explain `strike_distances`; the clips fit the reading that the five numbers are the upper ends of the bands a to e, in metres from the attacker.

| Skeleton | `strike_distances` | Attack clips | Weapon lands ahead (min / median / max) |
|---|---|---|---|
| Swordsman (sword and shield) | 1.40 1.50 3.0 3.5 4.0 | 11 | 0.80 / 1.87 / 3.01 m |
| Mace (one-handed) | not set | 7 | 0.64 / 1.48 / 2.79 m |
| Spear | 1.40 2.20 3.2 3.5 4.0 | 7 | 0.92 / 2.01 / 2.11 m |
| Two-handed sword | 1.15 1.75 2.10 3.0 3.5 | 10 | 0.88 / 1.71 / 2.76 m |
| Two-handed axe | 1.15 1.8 2.8 3.5 4.0 | 7 | 0.94 / 1.55 / 3.01 m |
| Pike, halberd | not set | 2 | 3.77 / 3.86 / 3.95 m |

  Eight of the eleven sword clips are in band c and land 1.6 to 2.9 m ahead. A sword fight in M2TW is therefore fought at about 2 m, not at arm's length.
- Already on record from the engine research (devlogs 0036, 0120, 0121): every soldier keeps his formation slot all battle. Under an attack he runs a melee action with one target, an attack position of his own, the chosen clip, and the bits `crowded`, `isSideStepping` and `allowAttack`; a man who cannot reach his attack position stands ready, waits and sidesteps for room. Sideways moves are the shuffle clips at 0.75 to 0.9 m/s.

## Reading the two observations against the data

A rough reading of `eng_early_spearman.png`: neighbours in the front rank stand about 0.7 of a body height apart, which is 1.2 m for a man of about 1.75 m (the M2TW skeleton's thigh is 0.46 m and shin 0.40 m, devlog 0120; his full height was not measured). That agrees with the unit file.

The same ratio for flanks, the gap between slot centres over the soldier's height:

| | Slot pitch | Soldier | Pitch in body heights |
|---|---|---|---|
| M2TW close order | 1.2 m | about 1.75 m | 0.69 |
| M2TW loose order | 2.4 m | about 1.75 m | 1.37 |
| flanks 0.2.1, normal | 1.4 m | 1.0 to 1.1 m | 1.27 to 1.40 |
| flanks at 1.64, normal | 1.4 m | 1.64 to 1.80 m | 0.78 to 0.85 |
| flanks at 1.64, wall | 1.05 m side to side | 1.64 to 1.80 m | 0.58 to 0.64 |

And the fight:

| | Distance between two men fighting | In body heights |
|---|---|---|
| M2TW, sword | clips land 0.8 to 3.0 m ahead, median 1.87 m | about 1.07 |
| M2TW, spear | median 2.01 m | about 1.15 |
| flanks 0.2.1 | a man stops 1.2 to 1.6 m from his enemy (`SEEK_STOP_SPREAD`), enemies rest 1.4 m apart | about 1.3 |
| flanks at 1.64 | the same metres | about 0.8 |

How tight the press gets in flanks: the log's `nn min/avg` column (each man's nearest neighbour of either side, `tmp/runs/scale/s164.log`) reads 1.32 m on average while the armies stand, 1.20 m at first contact, then 1.00 and 0.85 m as the lines press, with a minimum near 0.6 m. That is 0.8 of a body height at 0.2.1's size and 0.5 at 1.64.

So, by the numbers:

1. Standing, 0.2.1 looks like M2TW's loose order, which is what Gota said in the first prompt of this session ("wide like m2tw's loose formation"). At 1.64 the standing ranks are a little wider than M2TW's close order, not tighter.
2. Fighting, 0.2.1 is close to M2TW: its men come to a little over one body height from their enemy, and under 1 once the lines press. At 1.64 the same fight is a third closer than M2TW's and the press packs men to half a body height apart. This is the part that felt wrong, and it is the fight, not the formation: Gota's 1.64 screenshots that looked too tight (`ground_spot_200k.png`, `gap_clip_situation.png`) are lines in contact.
3. Observation 1 as written ("the current spacing is too tight", "even pikemen spearwall have gaps wider") is not what the numbers say for standing ranks. Still to check with a like-for-like picture: a flanks unit standing unengaged at 1.64 beside `eng_early_knight.png` at the same framing. Our soldiers may also read wider than M2TW's for their height (shields, levelled swords), which fills the gaps to the eye.

## Correction, the same day: the soldier's width

Gota's reply to the reading above: the gap between allies in our fight at 1.64 is clearly too tight to the eye, and the table made it sound close to M2TW's. Gota is right on both observations, and items 1 and 3 above are wrong as statements about what is seen. The measure was the mistake: slot pitch over body height ignores how wide the man is.

Measured from the shipped models (`assets/units/*.glb`, level 0, the pose they stand in, at the built 1.80 m):

| Model | Width with arms, shield and weapon | Torso alone | Weapon reaches ahead of his centre |
|---|---|---|---|
| Knight | 1.15 m (1.08 without the sword) | 0.45 m | 1.08 m |
| Man-at-arms | 1.07 m | 0.47 m | 1.10 m |
| Spearman | 1.00 m | 0.57 m | 0.29 m (spear upright) |
| Archer | 0.79 m | 0.52 m | 0.85 m |

What is left between two neighbours is the pitch less that width:

| | Between centres | The man's width | Free between neighbours |
|---|---|---|---|
| flanks 0.2.1, standing | 1.4 m | 0.66 to 0.70 m | about 0.7 m |
| flanks at 1.64, standing | 1.4 m | 1.08 to 1.15 m | about 0.3 m |
| flanks at 1.64, lines pressed | 0.85 to 1.0 m | 1.08 to 1.15 m | none: they overlap by 0.1 to 0.25 m |

And the sword of a man at 1.64 points 1.08 m ahead while the rank in front stands 1.4 m ahead, or 1.0 m in the press, so it ends inside that man. In Gota's M2TW screenshots every man has clear ground on both sides, standing (`eng_early_knight.png`) and fighting (`knight_attack-order_spacing.png`). M2TW's men also hold the shield in front of the body and the weapon close, so they are narrow across the rank. Ours stand square with the shield out at the side and are more than twice as wide as their torso. M2TW's own widths are not measured yet; the probe below reads the engine's collision radius.

So two things make our fight too tight at true size: the distances (the press, and a fight at 1.2 to 1.6 m where M2TW's is about 2 m) and the width of our standing pose.

## The soldier probe

Gota chose the live probe. It runs on this machine as the July one did (devlog 0057): a Steam launch shim starts it inside the game's container, it reads the game's memory, nothing is installed. `medieval2.exe` and `kingdoms.exe` are the same file in this install (same md5), so the plain game works and the battle struct is where July found it.

Files, all in `work/research/m2tw-extraction/`:

- `m2tw_soldier_probe.py`: every unit of both sides, then every soldier through the unit's soldier array. Per soldier per sample (2 a game second, `M2TW_PROBE_HZ` changes it): position, height, facing, his formation slot and his distance from it, two status words. Per unit per sample: name, side, soldiers left, and the 64 bytes that hold its action, melee state and live formation spacing. Once per battle: each unit type's spacing, mass, width and height from the unit table, each soldier's collision radii, and raw dumps of a few structs so offsets can be re-derived afterwards.
- `test_soldier_probe.py`: the parse chain run against a made-up memory image, since the probe cannot be tried without the game. It covers the layout as the M2TWEOP header gives it, a shifted soldier array with no self pointer, a stale pointer in a dead man's entry, and the sampling loop on a running clock. All pass.
- `m2tw_soldier_launch.sh`: the shim. Unlike July's it starts the game as Steam asked, with no mod and no other exe.
- `soldier_spacing.py`: from a capture, per unit over time: the distance to the nearest comrade (10th percentile, median, 90th), the median distance to the nearest enemy, the share of men with an enemy within 3 m, the mean distance from the slot; a plot, and maps from above at chosen times. Tried on a made-up capture.

Offsets come from `work/research/archer-evidence/eop_unit.h`. Anchored by the header's own offset comments: the unit's soldier array at +0x5F4 (the field after it is marked 0x5FC), its formation block at +0x1E40 to +0x1E80, the unit table's spacing at +0xF4 and width at +0x120, the soldier's position at +0x18. Not anchored, so found by search at run time and logged: the soldier's pointer back to his unit (expected +0x4DC), from which his slot (0x13C before) and status bits (0xC after) are taken. If those two come out wrong the positions are still right, and the raw dumps let the offsets be fixed without another play session.

Installed copies: the probe in the game folder, the shim at `/data2/SteamLibraryFlatpak/m2tw_soldier_launch.sh`. The captures land in the game folder as `soldiers-<stamp>.*`, about 230 KB a second for 2,600 men.

To run: Steam launch options for Medieval II set to `/data2/SteamLibraryFlatpak/m2tw_soldier_launch.sh %command%`, then a custom battle: stand 15 s after Start Battle, one unit to loose order for 15 s, knights attack for a minute, spearmen moved into the middle of an enemy unit and then ordered to attack for a minute. Clear the launch options afterwards.

## The first capture: M2TW's spacing, measured

Gota played one battle with the probe the same afternoon. It worked on its first run: the battle struct through July's pointer, the soldier array at +0x5F4, the soldier's unit pointer at +0x4DC as the header predicted. 680 men, 431 samples, 215 s of battle clock. Copy of the capture and its log: `work/research/m2tw-extraction/capture-20261003/`. Figure: `work/notes/vis/027-m2tw-fight-spacing.png` (`capture-20261003/figure.py` draws it).

What the capture holds: Gota's army only, five units of Dismounted Feudal Knights (120 men each) and the general's Feudal Knights (80). The enemy army was not in the side lists at the header's offsets. The probe now also looks for armies by their shape (`find_armies`), tested on the made-up image; the next capture should hold both.

Two things learned about the engine's data:

- The living are the first N entries of a unit's soldier array, N being the unit's own count. When a man dies the last living pointer takes his place, and the entries past N are stale copies of living men. The deceased bit of the header did not match the counts and is not used. `soldier_spacing.py` follows the count.
- Two status bits read as the header says: "in formation" (bit 27 of the first word) and "in combat" (bit 13 of the second).

The engine's own sizes, the same for every man of these units:

| | Value |
|---|---|
| Collision radius | 0.4 m, so a man is 0.8 m across |
| Height | 1.7 m |
| Selection marker radius | 1.0 m |
| Unit table: spacing close / loose | 1.2 / 2.4 m, read from memory as from the file |
| Unit table: width, height, mass | 0.4, 1.7, 1.2 (cavalry mass 1.0) |

The distance from each man to the nearest comrade of his own unit, median over the unit:

| Situation | Nearest comrade, median | 10th to 90th percentile | From |
|---|---|---|---|
| Standing, close order | 1.0 m | 0.8 to 1.2 m | units 0 to 3, 5 to 40 s (1.05, 1.02, 0.98, 1.01) |
| Standing, loose order | 2.1 m | 1.75 to 2.3 m | unit 4, 30 to 45 s |
| Moving under orders, not fighting | 0.75 to 0.85 m | 0.56 to 1.2 m | unit 0, 58 to 135 s |
| Fighting, settled | 1.5 m | 1.2 to 2.2 m | units 1, 2, 4 after 150 s; plateaus 1.55, 1.44, 1.58, and 1.47 for unit 3 |

The slots are 1.2 m apart with each man a little off his slot, so the nearest of his neighbours is about 1.0 m away.

How a unit opens once it fights, from the first sample with most of its men in combat:

| Unit | at contact | +10 s | +20 s | +30 s | +45 s | +60 s | +90 s |
|---|---|---|---|---|---|---|---|
| 1 | 0.98 | 0.95 | 1.23 | 1.26 | 1.43 | 1.47 | 1.58 |
| 2 | 0.95 | 1.22 | 1.29 | 1.33 | 1.42 | 1.44 | 1.31 |
| 4 (came in loose order) | 1.10 | 1.39 | 1.47 | 1.48 | 1.57 | 1.67 | 1.55 |
| 3 (second time, after breaking off) | 0.90 | 1.16 | 1.33 | 1.39 | 1.46 | 1.44 | 1.53 |

What that says:

1. Gota's observation holds, with numbers. A fighting unit stands half again as open as a standing one: 1.5 m between comrades against 1.0 m, with 0.7 m of free ground between bodies against 0.2 m.
2. It opens gradually: most of the way in 30 s, settled in 45 to 60 s.
3. The fight's spacing does not depend on the formation's. Unit 4 arrived in loose order at 2.1 m, was squeezed to 1.1 m at contact and settled at 1.5 to 1.6 m like the others.
4. Moving men bunch until their bodies touch: 0.8 m is exactly two collision radii. Unit 0 moved about for two minutes at 0.75 to 0.85 m and never fought.
5. Unit 3 shows the whole cycle Gota described. Fighting at 1.4 m, it broke off at about 115 s (the share in combat fell from 97% to 2%) and closed to 0.85 m within 10 s; fighting again from 135 s it re-opened to 1.4 m over 30 s.

Gota's account of the battle, given after the reading above, and it matches: nothing for the first 16 s; then about 15 s of loose order on the unit at the far left (unit 4, which then attacked as it was); about a minute of plain fighting; and the move order followed by an attack order on the fourth unit card (unit 3; screenshot `refs/unit_scale_1.64/probe/run0_move->attack.png`, the cards reading 121, 96, 80, 55, 74, 80 in the capture's unit order). Under the move order unit 3 closed from 1.41 m to 0.86 m in about 5 s and lost 27 men in 20 s inside the enemy (105 to 78). After the attack order it read 0.90 m, then 1.08 at +4 s, 1.17 at +12 s, 1.33 at +20 s and 1.42 at +32 s.

Against flanks, as the distance between comrades over the soldier's height:

| | Height | Standing | Fighting |
|---|---|---|---|
| M2TW | 1.7 m | 1.0 m, 0.6 of a height | 1.5 m, 0.9 of a height |
| flanks 0.2.1 | 1.0 to 1.1 m | 1.3 m, 1.2 to 1.3 | 0.85 to 1.0 m in the press, 0.8 to 0.95 |
| flanks at 1.64 | 1.64 to 1.80 m | 1.3 m, 0.75 | 0.85 to 1.0 m, 0.5 to 0.6 |

The flanks rows are the `nn` column of the game's log, which takes each man's nearest neighbour of either side, so they are not the same measure as M2TW's nearest comrade; the comparison is good to the first digit.

So 0.2.1's fight is at M2TW's spacing for its soldiers' size, which is why it felt right, and its standing ranks are twice as open as M2TW's close order. At 1.64 the standing ranks are near M2TW's and the fight is far too tight. The two games also move in opposite directions: M2TW's units open when they fight, ours close.

Still missing: the distance from a fighting man to his enemy, which needs the enemy army in the capture.

## What followed from the numbers

- The proposal: `docs/plans/021-room-to-fight.md`, figure `work/notes/vis/028-fight-room-proposal.png`. A man fights at his weapon's length, a swing needs a clear arc, and the standing pose gets narrower. Gota approved rule 1, rule 2 for a swing with its terms defined, and the pose.
- Ground per man in the capture: 1.3 to 1.4 m2 standing in close order, 5.5 m2 in loose order, 4.5 to 6 m2 fighting. A fighting unit covers about four times its standing ground, mostly in depth (unit 1: 21 by 6 m standing, 28 by 16 m fighting). 200k men all fighting need about 1 km2; today's field is 0.79 km2.
- M2TW's skeleton, from the animation files (`work/scripts/m2tw_anim/render_pose.py`, sheets in `refs/animation/m2tw_pose_refs/`): the joints span 0.68 m across at ease and 0.54 m across in the ready stance, which is side-on and 0.80 m front to back (spear: 0.65 and 0.55 m across, 0.93 m deep).
- Gota's order: the bigger map lands first; the scale and the room to fight are tried on a branch and merged after it; the pose merges on its own.
- Handoffs for Astra: `work/handoffs/085-narrow-stance-and-guard-for-astra-2026-10-03.md` and `086-big-map-for-astra-2026-10-03.md`.
- The height: 1.80 m was only the models' built size. Proposed 1.75 m to the helmet top, about 1.70 m bare-headed, which is M2TW's 1.7 m and the medieval average.

## Why M2TW opens up in a fight

Not proven; it is the reading that fits the data and Gota's third observation. M2TW has no spacing number that switches with the situation. Under a move order every man follows his slot, 1.2 m from the next, and only body collision keeps men apart, so a unit pushed into the enemy stays a tight clump. Under an attack order every man leaves his slot for an enemy of his own and walks to the spot his chosen attack needs, about 2 m from that enemy. Men in the middle have comrades in the way: they are the `crowded` ones, they cannot attack, and they sidestep with the shuffle clips until they find room. The unit therefore spreads from the rim outward over some seconds, which is what `spearman_then-attack-order_spacing-widened.png` shows.

flanks already has the same parts in its footwork (go to an enemy he sees, stop short of him, wait behind a comrade, sidestep in some windows). The differences are the distances, which are 1.2 to 1.6 m where M2TW's clips want about 2 m, and that nothing gives a fighting man room from the comrade beside him beyond the 1.4 m soft push.

## Not known yet, and how to find it

- How far apart comrades stand in an M2TW fight, how far a fighting pair stands, and how many seconds the spreading takes.
- The engine's default collision radius.

Ways to get them:

1. Top-down screenshots. The camera straight down over a standing unit, a unit fighting under an attack order, and a unit moved into the enemy and then ordered to attack. The discs give each man's position, and a standing unit in the same frame gives the ruler (1.2 m). About an hour to measure, good to about 10%.
2. The live probe (`work/research/m2tw-extraction/m2tw_probe.py`, devlog 0057) extended from unit morale to soldier positions. It reads the running game's memory from Linux with nothing installed. It needs the soldier array's layout from the M2TWEOP headers, found by search as the battle struct was, since the published offsets do not fit this exe. Half a day to a day of work, then ten minutes of Gota's play. Gives exact distances and the time course.

## Two facts asked on the way

- The field today is 1024 m wide and 768 m deep (`terrain.rs`: 16 by 12 chunks of 32 cells of 2 m).
- Trees and shrubs are true size. The game log gives the oak at 13.2 m, the mature oak at 13.7 m and the leaning oak at 9.0 m, planted at 0.85 to 1.10 of that; shrub A was built about 1.7 m tall (devlog 0155). A 0.2.1 soldier is 1.0 to 1.1 m.

## Open

- The scale branch waits. The fight's distances have to be settled first: at true size they need to grow toward M2TW's 2 m, and that is footwork, which is feel-critical. The standing pose's width is the other half.
- A second capture with the enemy army in it, for the distance between a fighting man and his enemy.
- What flanks does with these numbers: the fight's distances and the standing pose's width, each its own proposal.
