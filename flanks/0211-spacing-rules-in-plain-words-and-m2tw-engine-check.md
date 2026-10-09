# 0211: the spacing rules on the branch in plain words, M2TW's engine checked online, and what caused the robotic wave

Written by Claude Fable 5.1. 2026-10-07, the thread after handoff 099, on `feat/soldier-scale` at 9518d48 (26 commits on main 66044d1, not pushed). Nothing built in this devlog; it is a record, a figure, a research pass and one check in the port.

## What Gota asked

Gota, on the list of spacing layers in the first reply of the thread: "these are confusing, explain in plain terms without Claudish, and mark the ones we changed on this branch. Use visualization images if needed to explain each item (maybe you should, for documentation purpose too)".

On the widening: "I thought we use 1.4m or something for the spacing during combat. I think there should be another distance between soldiers who are not in the move order state, a slightly wider one than FL_BODY, to be used when engaging enemy unit when entering combat from idle/standing state etc. or via the attack order, like m2tw. I think we should research online just in case about how relevant physics engine of m2tw is like. From my playing experience, I feel like there's another combat circle than FL_BODY equivalent, but that's a hypothesis."

On the soft wall against the shared speed: "but the cause of robotic wave was the leg physics like how it's acceleration was superhuman etc."

## The nine rules, in plain words

Figure: `work/notes/vis/041-spacing-rules-on-the-branch.png` (script `work/scripts/viz/spacing_rules_041.py`). One panel per rule, top-down and to scale, with an orange tag where this branch changed it. All of it is in `src/sim/soldier.rs` unless said; the panel numbers below are the figure's.

1. **His mark in the block.** Every man has a mark in his regiment's block, 1.4 m from the next, 1.05 m in a wall. More than 0.7 m off it, he walks back; once he steps, he finishes the step. In a fight he never steps backward to his mark, and he waits where he is when a comrade stands between him and it. Under a Move order the whole block of marks moves and every man follows his own. *Branch: the pitch became a value per kind, `FL_GAP` (c2c6bb7), still 1.4 m for all. Code: `drive`, `hold_the_frame`, `src/formation.rs`.*
2. **The spring between any two men.** Any two men closer than 1.4 m push each other apart, harder the closer they are; a heavy man pushes a light one further. A standing man ignores a small push: one neighbour at 1.3 m does not move him, one at 1.1 m does. In a packed crowd the push is cut off together with his drive (rule 4), so a press does not shake. The push is not legs and keeps the old acceleration cap. *Not changed. Code: `scan` (`SEP_RADIUS`, `SEP_STRENGTH`), `steer` (`STAND_GRIP`).*
3. **The body distance, `FL_BODY`.** Closer than this, the pair is moved apart by hand: 15 % of the overlap each tick, on every tick, the heavier man less. 0.2.1 took 30 % and only when the sum was over 1 cm, so it came in jumps, and six bodies at 30 % threw a man past their middle and back: the twitch. A standing man's lean, arms and shield take the first 10 cm of a squeeze before he gives ground to it. Nothing wider than this circle is enforced in a press. *Branch: 0.9 to 1.0 m (a8dbdf6), allowed up to 1.4 m (a93080a), 15 % and no 1 cm threshold (a6dbe68), the stance (2f81fe6). Gota's default 1.2 m is not set yet. Code: `body_distance`, `CORR_GAIN`, `STANCE`.*
4. **The crowd brake.** The tighter his neighbours press him, the less of his own drive he keeps: all of it with six men at 1.3 m (arm's length), none with six at 0.9 m (his own width); the spring fades with it. So a packed mass stops shoving itself tighter. The two distances follow the men's real size (`FL_UNIT_SCALE`), not `FL_BODY`. 0.2.1 had 1.12 to 0.82 m. *Branch: tied to the body distance first (e8840e0), then set by the men's size (8bf1152), with `FL_BRAKE=body` to replay the first. Code: `crowd_brake`, `yield_to_crowd`.*
5. **What his legs can do.** His own legs accelerate at most 4 m/s², about what a man in armour with a shield manages from a stand. 0.2.1 had no cap: 28 m/s² at every start and stop, three g, so the whole lane behind an opening gap surged and stopped as one body, the stop-and-go waves. Only the steering toward his wish is capped; the pushes between bodies keep the old cap. *Branch: new (8bf1152), `FL_LEG_ACCEL`. Code: `leg_accel`, `steer`.*
6. **Running into a body: the soft wall.** A man who runs into a body sinks up to 8 cm into it (his own 6 to 10 cm) while his legs lose their speed toward it, and goes no further. The man he hits is handed none of his speed; he moves only as the setting apart (3) and the spring (2) move him. 0.2.1 stopped the runner dead on the touch tick and he walked in again the next: the twitch. The shared speed tried before (a6dbe68, now `FL_CONTACT=share`) handed half the runner's speed to the man hit, and that speed hopped from row to row: the lurch. *Branch: new (2f81fe6). Code: `GIVE`, `give`, `steer`.*
7. **Where he fights from, and the lane.** He fights at 0.9 of his weapon's reach: 1.8 m from his enemy for a man-at-arms (reach 2.0), 1.62 m for a knight (1.8), 2.16 m for a spearman (2.4), 1.44 m for a bowman's blade (1.6); 0.2.1 closed to 1.2 m for all. Out of formation, with open ground behind him, he steps back out to that distance from an enemy who has come inside it. He does not strike through a comrade in the 0.7 m wide lane between him and his enemy: he waits, and sidesteps in a fifth of the one-second windows. Devlog 0182 measured that these two thin the contact and do not open a mass. *Branch: new (b34d73c, the lane f4f2d7d). Code: `close_in`, `FIGHT_SHARE`, `in_lane`.*
8. **Under a Move order, an attack, or idle.** Rules 2 to 6 are the same under every order. The one difference: a man under a Move order closes on no enemy and walks his mark straight through them, as M2TW's unit packed to 0.86 m inside the enemy under its Move order. There is no wider distance for a man who fights from idle or under an attack, and nothing that opens a packed unit over seconds once its order ends. The spring would do it at once, but it is cut off in a press and a standing man ignores its small pushes. *Not changed. This is where the widening would go.*
9. **M2TW, for comparison.** Below.

Also on the branch and not a spacing rule: every soldier is drawn at his true height (aa0235b), which is why a 1.0 m press now shows as a press where 0.2.1's 0.9 m did not (devlog 0182, figure 032).

Gota's "1.4 m for the spacing during combat": the 1.4 m is the mark pitch (rule 1) and the spring's rest distance (rule 2). Both hold a man in the open and both give way in a press: the spring is cut off by the brake and the marks are not dressed while engaged. The measured distances are in handoff 096: standing 1.31 m, a settled fight at 20k 1.25 m, the 200k mass 0.87 to 1.04 m.

## The widening as Gota now describes it

A second distance for men not under a Move order, a little wider than `FL_BODY`, used when a unit meets the enemy from idle or standing or under an attack order. Not built; nothing on the branch differs by order except rule 8's "closes on nobody". It is the open design of handoff 099 with one more detail: Gota's circle sits between `FL_BODY` and 1.4 m, not at 1.5 m.

## M2TW's engine, checked online

Read on 2026-10-07. The forums (totalwar.org, twcenter.net, the Steam guides) refuse the fetch tool; what came from them is the search engine's snippet only. The two sources read in full are official or struct-level:

- Feral Interactive's Rome Remastered modding documentation, the EDU guide (`documentation/data_file_guides/EDU.md` in `github.com/FeralInteractive/romeremastered`). Same engine family as M2TW. The `soldier` line is `unit_model, soldiers, extras, mass (,radius,height)`: **mass** is the collision mass, "units with big mass values can push their enemies harder and break through enemy lines easier and also hold against enemy pushing better; the mass ratio is not fixed"; **radius** is a hidden attribute, default 0.4, "the area surrounding each single soldier that he occupies as the engine perceives it; small radius makes a unit fight better, in that it allows soldiers to fight more closely to each other"; **height** hidden, default 1.7. The `formation` line is the close and loose spacing side to side and front to back in metres, the ranks and the formation types. No other spacing or combat distance is documented anywhere in the guides (grep over all of them for radius, collision, spacing, push).
- The M2TW Engine Overhaul Project's reverse-engineered structs (`github.com/EOP-Labs/M2TWEOP-library`, `types/unit.h` and `types/battle.h`, cloned and grepped). Per soldier (`soldierInBattle`): his formation mark `formationX`, `formationY` and `formationDisplacement`, an `isInFormation` bit, `inCombat`, `targetCount` (how many enemies target him), `lastEnemiesCollided`, `lastCollision`, `momentum`, `brace`; his locomotion (`locomotiveElement`): `destinationRadius`, `stoppingDistance`, `stoppingModifier`, `speedModifier`, **`radiusModifier`**, `cone`, `directionOffset`, `blockedCounter`, `deflectionCounter`. Per unit: the EDU's four spacings (`unitSpacingFrontToBackClose` and so on), a live `unitSpacingX`, `unitSpacingY`, `xRadius`, `yRadius`, and the state `formationMovingThrough`. The melee action (`actAttackMelee`): a target unit and soldier, an attack position `posX`, `posY` of his own, `crowded`, `readyStance`, `allowAttack`, `isSideStepping`, `attackContact`, a `hintSoldier`; its context holds up to 100 enemies and 100 friendlies, each with a `collided` flag, and a `blockedCounter`. Nothing in EOP's Lua binding exposes `radiusModifier`; nothing in its code sets it.

What this says about the hypothesis:

- There is one body circle per man in the data, radius 0.4 m (bodies touch at 0.8 m), and it is soft: the capture of 2026-10-03 had a third of marching men under 0.8 m (devlog 0208). Mass sets who pushes whom.
- There is no second spacing number for combat in the files or the documented fields. The engine does keep a per-man `radiusModifier` on his locomotion, a multiplier on that one circle, and nothing read says when it is set. It could be a widened circle in some state, or a shrunk one for passing through, or the mount's ellipse; unknown.
- What makes a fighting unit open up, by everything read: each attacking man walks to an attack spot of his own, set by his clip's strike distance (1.4 to 4 m from his enemy, devlog 0177), a crowded man does not strike and sidesteps for room, and every man keeps his formation mark as the place he returns to. So the spread Gota sees under an attack order is the men going to their spots and the crowded ones stepping aside, not a wider collision circle. Gota's own M2TW capture fits: 0.90 m under the Move order, then 1.08, 1.17, 1.33, 1.42 m at 4, 12, 20 and 32 s after the attack order (devlog 0177).
- Checks that would settle the `radiusModifier` question, both from Gota's install: read it per soldier in a live capture with the probe of devlog 0208 (the field's offset is in EOP's `unit.h`, the locomotion block at `soldierInBattle + 0x124`), under a Move order and then under an attack; or set the hidden EDU `radius` of one unit to 0.8 and watch whether its men still open up under an attack (if they open the same, the spread is not the circle).

The snippets from the blocked forums agree with the above (the hidden radius 0.4, mass pushes, a smaller radius means more men fight) and add nothing on a combat circle. Sources: [Feral's EDU guide](https://github.com/FeralInteractive/romeremastered/blob/main/documentation/data_file_guides/EDU.md), [M2TWEOP library](https://github.com/EOP-Labs/M2TWEOP-library), [Steam guide to unit types](https://steamcommunity.com/sharedfiles/filedetails/?id=909957531) (snippet), [Collision in M2TW, totalwar.org](https://forums.totalwar.org/vb/showthread.php/73888-Collision-in-M2TW) (not readable), [Mass and push back, twcenter](https://www.twcenter.net/threads/mass-and-push-back.552296/) (not readable).

## What caused the robotic wave: the legs or the shared speed

Gota's claim: the robotic wave came from the legs' superhuman acceleration. The port's `rear` scene (a unit ordered onto the place of a unit that holds, into its back, at `FL_BODY` 1.2; the measure of devlog 0210: the share of the holder's men stepping, smoothed ground speed over 0.3 m/s, over seconds 3 to 8, the peak, and how far the holder is carried by 20 s), run on 2026-10-07 with the shared speed and the soft wall each with the legs uncapped and capped at 4 m/s², and with the old brake band and the current one. Script: this session's scratchpad `rear_legs.py`, 20 lines on `contact_toy.py`.

| Contact | Legs | Brake band | Stepping 3 to 8 s | Peak | Holder carried at 20 s |
|---|---|---|---|---|---|
| Shared speed | uncapped | 0.2.1's | 33 % | 48 % | 1.55 m |
| Shared speed | uncapped | 1.3 to 0.9 m | 22 % | 38 % | 1.40 m |
| Shared speed | 4 m/s² | 0.2.1's | 37 % | 47 % | 1.56 m |
| Shared speed | 4 m/s² | 1.3 to 0.9 m | 26 % | 38 % | 1.43 m |
| Soft wall | uncapped | 0.2.1's | 14 % | 22 % | 0.85 m |
| Soft wall | uncapped | 1.3 to 0.9 m | 5 % | 17 % | 0.57 m |
| Soft wall | 4 m/s² | 0.2.1's | 11 % | 17 % | 0.78 m |
| Soft wall | 4 m/s² | 1.3 to 0.9 m | 7 % | 17 % | 0.53 m |

Read:

- The leg cap does not touch the shared speed's wave: 33 % to 37 % stepping, the holder carried 1.55 m either way. The lurch is the speed handed from the runner to the man he hits, and from him to the next; his legs only set how fast he stops afterwards, and a capped man stops slower, not faster.
- The contact rule is the big lever: with everything else equal the soft wall has less than half the stepping and the holder carried about half as far.
- The brake band takes about a third off both. That is the part of the current build's calm that is not the soft wall.
- In the game, Gota's two verdicts were on builds with the same uncapped legs (a6dbe68 and 2f81fe6 both before the cap), so the game comparison isolated the contact rule too. The "ABS" look of the hold build (c1491d1) was each man cancelling his closing speed in one tick, which is a contact rule as well, not the legs.
- Not played: the shared speed with the cap and the band. `FL_CONTACT=share` on the current binary is exactly that, if Gota wants to see it.

## State

No code changed. New: figure 041 and its script, this devlog. The widening is still the open design; Gota's description above is the spec so far, and the M2TW reading above is the one thing added to it.

## 2026-10-08: the push scene with the shared speed under the leg cap and the current brake

Gota: the clips to see are the gap waves in the push scene (three blue units hold 2 m apart, a red mass of twelve is sent past them, deaths at the contact), not the arrival wave of the rear scene. And the wave fix did change the crowd brake: the soft wall build still had the band tied to the body distance, and the next commit set the legs' cap and the crowd brake's band by the men's size, the two causes of devlog 0210.

The port got two entries for it: `wall-legs` (the current build: the soft wall, legs 4 m/s², the band 1.3 to 0.9 m) and `share-legs` (the shared speed with the same cap and band, `FL_CONTACT=share` on the current build). Clips, 100 s, deaths five a second from 10 s, men at their drawn width, the four versions side by side: `tmp/runs/soldier-scale/contact-toy/push-deaths-drawn-share-legs-100s.mp4` (the whole scene) and `push-close-drawn-share-legs-100s.mp4` (the right gap). Figure 042, `work/notes/vis/042-gap-waves-shared-speed-with-legs-and-brake.png` (script `gap_waves_shared_speed_legs_042.py`): figure 040's space-time picture of the right gap's column for the four.

The blue line in that scene (its living men's mean movement along the push, and how many live), from `push_line.py` in this session's scratchpad on `contact_anim.run`:

| Version | Blue alive at 40 s | Line carried at 40 s | Blue alive at 100 s |
|---|---|---|---|
| Shared speed, as played | 315 | 6.08 m | 158 |
| Shared speed, legs 4 m/s², brake 1.3 to 0.9 m | 313 | 7.42 m | 165 |
| Soft wall, as played | 318 | 0.57 m | 176 |
| Current build | 318 | 0.61 m | 179 |

Read:

- Under the shared speed the holding line does not hold: the red mass, Move-ordered 60 m past it, carries it 6 m by 40 s and is through by 50 s. With the leg cap it is carried further, 7.4 m, because a man who is handed a speed now has only 4 m/s² of legs to cancel it with. In figure 042 both shared-speed panels are a solid dark band, the column pouring through at over 40 cm/s, and the line is gone by 50 to 60 s; there is no press left for a wave to run in.
- Under the soft wall the line holds (0.6 m), the soft-wall-as-played panel shows the stripes, the stop-and-go of the column every 1.5 to 2 s from 8 to 45 s, and the current build's panel is steady at 10 to 25 cm/s with no stripes, as in figure 040.
- So the leg cap and the band cure the waves on the soft wall and cannot be judged on the shared speed in this scene, because the shared speed loses the line first. The two contact rules are not interchangeable with the same cap and band.

## 2026-10-08: the branch's close-out begins

Gota's decisions after the discussion of the two contacts: keep both, the shared speed as the default (it moves and pushes things and looks more interesting; the soft wall is the safe one), a player toggle on the settings branch later; the combined rule (sink over the give, then hand on the excess with the feet taking a share) on another branch; the lane out; FL_BRAKE=body out and FL_LEG_ACCEL folded into its constant; FL_FIGHT_ROOM and main's old path out after fresh fingerprints from the final build; the step back kept or dropped by a count of how often it fires; the killing pace measured against 0.2.1 with one thing put back at a time before naming a cause; archers at true size; performance by the perf rules at the end; the widening, the settings and the module split of `soldier.rs` on other branches.

### Step 1: the body distance 1.2 and the shared speed as default (b92b974, cdc4227)

`BODY_DISTANCE` 1.0 to 1.2; `share_contact()` true unless `FL_CONTACT=wall`; the comments in `steer` and on `GIVE` describe the share as the rule and the soft wall as the alternative. Strict clippy clean, 35 tests pass. The chain (`tmp/runs/soldier-scale/shake/share-checks.sh`, log `share-checks.log`), every scene with the default and with `FL_CONTACT=wall`:

| | Shared speed (default) | Soft wall (`FL_CONTACT=wall`) |
|---|---|---|
| Fingerprints with `FL_FIGHT_ROOM=0` | direction 18 of 18, two-on-one 28 of 28 equal to main | (the same binary) |
| Six-on-one pile at 1.2: shakers; the press | 0 of 3,114; 0.97 to 1.13 m | 0 of 3,152; 0.97 to 1.13 m |
| Pile: the attacked front gave by 24 s, 40 s; victims left at 40 s | 5.1 m, 8.5 m; 194 | 3.7 m, 3.7 m; 223 |
| Pile at 1.0, 1.1: shakers | 0, 0 | |
| Through a friendly unit: pace; the holder's disorder at 12 s | 1.89 m/s; 2.75 m | 1.94 m/s; 1.85 m |
| Spearwall charge at 40 s: wall lane | 398 spearmen against 377 knights, the wall back 1.2 m | 402 against 378, 1.0 m |
| Push scene (`FL_TEST_PUSH=1`) at 60 s: line left; mass past it | 469 of 1,152; 1,362 of 1,362 | 416; 1,351 of 1,351 |
| 200k scripted front: shakers to 71 s; alive at 71 s | 0; 188,474 | 0; 188,593 |

Read: zero shakers everywhere under both. The share pushes: the enemy pile's front gives 8.5 m by 40 s against 3.7 m, and the pile kills faster (194 left against 223); a unit passing through a friendly one disorders it more (2.75 m against 1.85 m) at about the same pace. In the game's push scene the holding line is not carried as in the port: both contacts leave it at about the same place by 60 s, because the line's men fight and die and the mass goes through the gaps and round the ends, where the port's line only stood.

The 200k AI battle of the chain did not start: `FL_AUTOSTART=1` needs `FL_DEPLOY=0` to skip the picker and the deployment. Rerun with it; its numbers follow.

The 200k AI battle, rerun with `FL_DEPLOY=0` (one run each, not deterministic):

| | Shared speed | Soft wall |
|---|---|---|
| Shakers, 8 to 99 s | 0 | 0 |
| Alive at 99 s | 188,102 | 190,870 |
| Fighting units, comrade distance at 60 s, 100 s | 1.05 m, 1.12 m | 1.06 m, 1.04 m |
| Frame with a tick, p50 | 6.2 ms | 5.3 ms |

Both calm; the share kills a little faster and its press opens a little more as the battle goes on. Step 1 committed as b92b974 and cdc4227.

### Step 2: the lane out

`in_lane`, the `SIGHT_LANE` bit, the look for a comrade in the lane, the swing's hold and the closing footwork's wait with its sidestep, `Step.lane_blocked`, `src/lane_overlay.rs` with `FL_DEBUG_LANE` and its plugin line in `main.rs`. `SPEAR_LINE_HALF_W` stays for the spear's point. The sidestep for a man blocked on his way stays. Strict clippy clean. The chain reruns on it as `lane-checks.log`.

The chain on the lane build (`lane-checks.log`): fingerprints equal to main (18 of 18, 28 of 28); zero shakers in every scene under both contacts; the pile at 1.2 with the share: the front gave 5.3 m by 24 s and 8.8 m by 40 s, 195 left (step 1: 5.1, 8.5, 194); with the wall 3.7 and 4.7 m, 209 left (3.7, 3.7, 223); the pass 1.89 and 1.94 m/s as before; the push scene the same; the 200k front 187,557 alive at 71 s; the 200k AI battle 190,273 alive at 99 s with the share and 192,836 with the wall (not deterministic). So the lane was hardly ever the thing holding a blow back, as devlog 0182 had found. Committed as the commit after cdc4227.

### Step 3: FL_BRAKE=body out, FL_LEG_ACCEL folded into LEG_ACCEL

Strict clippy clean, 35 tests pass. Fresh fingerprints of the lane build on the default path (`tmp/runs/scripts/gates/lane-default-*`, the four scenarios: direction, archery, wide pile, two-on-one) against the same on this build: 18 of 18, 18 of 18, 28 of 28, 28 of 28 equal. Committed as the commit after afe113c.

### Step 4: how often the step back fires

A count in the room log (`STEP_BACK_TICKS` in `soldier.rs`, read every 5 s by `room_log`; committed as the commit after 545c854). Man-ticks per 5 s window, as men stepping back on an average tick (one man's step lasts about 10 to 15 ticks), with the shared speed at 1.2; logs `tmp/runs/soldier-scale/body/stepback-*.log`:

| Scene | 15 s | 25 s | 35 s | 45 s | 65 s | 95 s | Fighting regiments at the end |
|---|---|---|---|---|---|---|---|
| 200k AI battle | 10 | 28 | 38 | 35 | 23 | 19 | 60 |
| 20k AI battle (10,000 a side) | 9 | 24 | 26 | 25 | 21 | 11 | 16 |
| Six-on-one pile | 0.1 | 0.3 | 0.2 | | | | 4 |
| Direction test | 0 | 0 | 0 | 0 | | | 5 |

Read: in a press it never fires (a man needs open ground behind him); in an open fight a few dozen men at a time are stepping back, in the 20k battle as many as in the 200k one, since only the fighting rims count. Keep or drop is Gota's call; the branch is about fifteen lines and costs nothing when it does not fire. Steps 5 to 7 go on with it kept; dropping it later is that deletion.

### Step 5: FL_FIGHT_ROOM and main's old fight out

Deleted: `fight_room()`, `BODY_DISTANCE_OLD`, `CROWD_STOP`, `CORR_GAIN_OLD` with `corr_gain()`, `CLOSE_STOP`, `SEEK_STOP_SPREAD`, the 1 cm threshold on the setting apart, main's contact (the dead stop on the touch tick), the branches at the brake, the fight distance, the step back, the stance, the leg cap and the look-around, and `Step.corr_len2`, which only main's contact read. `CROWD_SLOW` stays: the far look uses it as the crowd above which a pressing man stops looking for a far enemy. `soldier.rs` 1,878 to 1,776 lines. Strict clippy clean, 35 tests pass. Fingerprints on the default path against the build before (`switches-out-*`): direction 18 of 18, archery 18 of 18, wide pile 28 of 28, two-on-one 28 of 28 equal. The chain's fingerprint step now compares against `switches-out-*`, since the comparison with main through the old path is gone with it.

### Step 6: the killing pace against main, measured first

`tmp/runs/soldier-scale/shake/pace.sh <name> [ENV...]` (FL_BIN picks the binary): the living count from the fingerprint log every 2 s in the six-on-one pile, the 200k scripted front and the 200k AI battle (one run, not deterministic). Main's binary is `tmp/runs/soldier-scale/bin/flanks-main` (66044d1). The build after 46bb464 at its defaults (body 1.2, shared speed):

| Alive | Main | Current build |
|---|---|---|
| Pile at 24 s, 40 s | 3,164; 2,914 | 3,239; 2,978 |
| 200k front at 40 s, 70 s | 192,406; 184,609 | 194,372; 187,779 |
| 200k AI battle at 60 s, 100 s | 188,971; 172,648 | 195,644; 192,121 |

Dead by the last report: the front 15,391 on main against 12,221 (the current build kills at 0.8 of main's pace there); the AI battle 27,352 against 7,879 (0.29 of main's pace). Trials follow, one thing at a time on the same binary and then on trial builds: the body at 1.0 (`FL_BODY=1.0`), the soft wall (`FL_CONTACT=wall`), the brake band at main's distances (1.12 to 0.82 m), the closing distance at 1.2 m for every kind.

Trials on the same binary (alive; the AI battle one run each):

| | Pile 40 s | Front 70 s | AI 100 s | AI dead by 100 s, as a share of main's 27,352 |
|---|---|---|---|---|
| Main | 2,914 | 184,609 | 172,648 | 1.00 |
| Current build | 2,978 | 187,779 | 192,121 | 0.29 |
| Body at 1.0 (`FL_BODY=1.0`) | 2,986 | 188,062 | 189,536 | 0.38 |
| Soft wall (`FL_CONTACT=wall`) | 3,021 | 187,908 | 192,326 | 0.28 |

Neither the body distance nor the contact is the cause: the pile and the front are within a few percent of the current build either way, and the AI battle stays at a third of main's pace. Next the trial builds: the brake band at main's distances, then the closing distance at 1.2 m for all.

Trial build, the brake band at main's distances (`BRAKE_FAR` 1.12, `BRAKE_NEAR` 0.82; the setting apart, the share, the cap and the fight distance as on the branch): pile 2,943 alive at 40 s (main 2,914, the branch 2,978), the attacked front gave 7.2 m; the 200k front 185,391 at 70 s (main 184,609, the branch 187,779); the AI battle 188,802 at 100 s (11,198 dead, 0.41 of main's). So the band's start at arm's length is the whole of the difference in the pile and the scripted front, and only a part of it in the AI battle. Next trial build: the closing distance at 1.2 m for every kind.

Trial build, the closing distance at 1.2 m for every kind (the band back at 1.3 to 0.9 m): pile 2,982 alive at 40 s (the branch 2,978), the 200k front 194,372 and 187,779, the same counts as the branch to the man (a Move-ordered line closes on nobody, so the stop distance never acts there), the AI battle 191,589 at 100 s (the branch 192,121). The fight distance by weapon is not the cause anywhere. Next, AI battle only: the band with the body at 0.9 m, the band alone a second time, and main a second time, to see the run-to-run spread.

AI battle only, 100 s (`PACE_ONLY=ai`), dead by 100 s:

| | Dead by 100 s |
|---|---|
| Main, run 1 and run 2 | 27,352; 22,086 |
| The branch (body 1.2, band 1.3 to 0.9 m) | 7,879 |
| The band at main's distances, run 1 and 2 | 11,198; 14,746 |
| The band at main's distances and the body at 0.9 m | 22,445 |
| The body at 0.9 m... at 1.0 m, the branch's band | 10,464 |
| The soft wall, the branch's band | 7,674 |
| The closing distance at 1.2 m for all, the branch's band | 8,411 |

The cause, named: the killing pace in the AI battle is set by how many men stand within reach of an enemy, and two of the branch's changes thin that: the crowd brake starting at arm's length, so an arriving unit eases off before it packs against the enemy, and the body distance of 1.2 m against 0.9 m, a press 1.7 times thinner by area. With both at main's values the AI battle kills at main's pace (22,445 against main's 22,086 and 27,352); each alone gives about half. The contact rule and the fight distance by weapon are not causes. In the pile and the scripted front the band alone is the whole of the difference. Main's own spread between two runs is 5,000 dead, a fifth, so single runs are read to that.

The old band in the port, figure 043 (`work/notes/vis/043-gap-waves-old-band-with-the-cap.png`, script `gap_waves_old_band_043.py`): the push scene's right gap column under the shared speed with the cap, the current band against 0.2.1's band, at the body 1.2 and at 0.9. The shared speed carries the holding line in that scene whatever the band; the band only sets how fast: the mass is through by 58 s with the current band, by 22 s with 0.2.1's, and at 0.9 with 0.2.1's band the column shows the lurch early (blue patches, men thrown back) and is through by 45 s. The stripes cannot be read under the share there because the flow saturates the scale. So in the port, 0.2.1's band under the share weakens a holding line further.

### Step 7: why archers kill 1.9 times faster at true size

The archery scenario (`FL_TEST_ARCHERY`, the gate's `arch`, alive from the fingerprint log): at 36 s main's binary has lost 146 men, the branch 279 (ratio 1.9, as devlog 0182 measured at the 1.0 body). The cause is in `src/arrows.rs`: an arrow hits a man when its path passes within `HIT_RADIUS` 0.35 m of his axis and within his half height above and below his centre. The branch draws the men at their true height, so `half_height` rose from about 0.55 to 0.85 m and the vertical window an arrow can hit nearly doubled; the horizontal radius did not change. The damage per hit (`missile::BASE_DMG` 20 times 1.28 to the power of attack minus armour and shield) was calibrated against archery-range logs with the small bodies: about 11 percent kills per arrow against unarmoured men, 4 against shielded lights, 1 against heavies.

A trial with the damage halved (`BASE_DMG` 10.5): 12 dead at 36 s against 279. Damage is a threshold against hit points (80 to 160), so halving it stops one-hit kills instead of halving them; the damage is not the lever. The kills per arrow scale with the hits per arrow, which the true height doubled. Proposal: M2TW's own mechanism for this, a lethality per landed arrow (the share of hits that wound; the rest glance off armour or the shield's rim), first set to 146 over 279, about 0.52, then checked per kind against the three calibrated shares. Not built; it waits for Gota's yes.

### Step 8: performance by the perf rules

`tmp/runs/soldier-scale/perf/perf-ab.sh`: 200k, `FL_TEST_FRONT=1` with the AI on, `FL_WINDOW=2560x1360`, the flank view locked from both ends, 120 s each, order main, branch, branch, main; the fps counter's 2 s reports averaged over the last 60 s; the load and the GPU clock logged beside each run (`tmp/runs/soldier-scale/perf/`). Main's binary is `bin/flanks-main` (66044d1), the branch `target/opt-dev/flanks` at 46bb464. Unattended: the window was not focused by hand and the desktop was not checked for other drawing, but nothing else was started.

| Run | fps | Sim step |
|---|---|---|
| Main, west end | 150 | 9.37 ms |
| Branch, west end | 167 | 9.42 ms |
| Branch, east end | 157 | 9.29 ms |
| Main, east end | 139 | 9.41 ms |

The branch reads 12 to 13 percent higher from both ends at the same sim step; the GPU clock sat at 1,800 to 1,950 MHz in all four (a 600 MHz idle reading at the first run's start). The load average ran 6 to 12 across the runs, the game's own threads. The wider spacing draws fewer men in the near levels from the flank, which is the likely reason; not isolated.

### The figure redrawn

Figure 041 redrawn for the state at 46bb464: the body at 1.2 m in panel 3, the shared speed as the rule in panel 6 with the soft wall as the alternative, the lane gone from panel 7 with the step back's count in its place. The panels in `work/notes/vis/041-panels/` are recropped (`041-6-shared-speed.png`, `041-7-fight-distance.png`).
