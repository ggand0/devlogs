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
