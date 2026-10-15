# 0241 Block and parry: the sim core

Written by Claude Fable 5.1.

Date: 2026-10-13. Branch `feat/parry-and-block` from main 58f90d3. Plan 024, research in devlog 0240. This entry: what was built, the first numbers, what is open.

## What changed

- **The stats decide whether a blow lands, not how hard.** `sim/damage.rs`: every blow rolls M2TW's hit curve (`land_chance`) on attack minus defence by sector, times a hit rate (1.75, `FL_HIT_RATE`). A landed blow deals its weapon's flat damage (`BASE_DMG` 18 sword, 20 spear, the median matchup's damage before). The damage factor (`FACTOR_MULT`, `FACTOR_CLAMP`) is gone from melee; the arrows keep their own curve and clamp.
- **The sectors are M2TW's.** Skill counts in full from the front, half from the sides, not from the rear; the shield from the front and the left; armour everywhere; the open side gives the blow 5 points and the back 10.
- **A blow that does not land is credited** to the shield, the skill or the armour in proportion to their points, for the sound and the counts (`OUT_*`), the way the sound was drawn before.
- **The warning.** When a man starts a melee wind-up, his chunk emits a `DamageEvent` with `warn` set to the ticks until the strike. The serial apply reads it: the victim answers if he is free (ready, or in his recovery past a wind-up's length since his own blow, not staggered, not already answering) and sees the attacker within 90 degrees, and his regiment is not routing.
- **The answer** is the new swing state `SWING_DEFEND` (state value 3), the style bits naming block or parry, `target` the attacker, `swing_t` a wait plus the clip. A man with a shield meets a blow from the front or his left with the shield unless he draws the parry (`parry_share`: knights 0.5, men-at-arms 0.35, spearmen 0.2); the open side and men without a shield parry. On time with the share `defence_skill x GUARD_PER_SKILL` (0.10, `FL_GUARD`), less the fatigue attack penalty: a knight 6 in 10, a man-at-arms 4, a spearman 3, a bowman 2. On time means the guard is up at the strike tick; late means it rises as the blow arrives.
- **At the strike**, before the roll: a victim whose guard is up (block: raised in 6 ticks, up 15, down 6; parry: 6, 6, 9) facing the attacker within 45 degrees stops the blow, whatever the points. A parry that stops the blow **counters at once**: his swing byte becomes a wind-up at half his kind's wind-up, aimed at the attacker. A parry not yet up has his guard open: no skill points against that blow. Impalements are never stopped.
- **The stumble** of a landed, non-killing blow is a flat 0.65 (the stats have already spoken in the roll); a man stumbled out of an answer loses it.
- `DirTestStats` counts outcomes per victim team; the direction test and the arena logs print them.
- The kernel: a `SWING_DEFEND` branch (feet planted, eyes on the attacker, the timer); the out-of-formation rule ignores the defend state (a man who blocks stays where he is); the warning pushed at wind-up start.

Not changed: the render and the audio clips. A defending man shows the ready pose; the counter shows as a short stab; the stopped blow plays the shield or the parry steel sound. The clips and the code poses are the next part.

## The numbers against main

Main is 58f90d3, built from a `git archive` export in `tmp/runs/parry/main-src` (binary `tmp/runs/parry/flanks-main`); the same scenes, the same seconds, logs in `tmp/runs/parry/`.

| scene | main | branch |
|---|---|---|
| Direction (`FL_TEST_DIR=1`), 42 s, kills front / side / rear | 451 / 210 / 685 | 186 / 133 / 300 |
| Direction, victims alive of 500: control / test / lone | 4 / 0 / 150 | 304 / 239 / 338 |
| Arena (`FL_ARENA=1 FL_ARENA_AUTO=1`), 40 s, frontal lane alive | 454 vs 445 | 496 vs 495 |
| Arena, rear lane alive | 500 vs 356 | 500 vs 462 |
| 200k scripted front (`FL_TEST_FRONT=1 FL_UNITS=100000`), dead at 75 s | 12,897 | 2,084 |

So the direction scene kills 2.2 times slower, the frontal duel lane 11 times slower (9 men lost against 101), the rear lane 3.8 times slower, the 200k front 6.2 times slower. The rear's share of the direction kills is the same (51 and 48 percent); the frontal duel is where the guard and the roll both bite.

Branch detail, direction scene at 42 s (blue victims): blows 5,022 front, 1,192 side, 2,180 rear; damage per blow 4.4 / 9.2 / 16.6 of 18, so 24 percent of frontal blows land, 51 from the side, 92 from behind. Outcomes: landed 3,847, blocked 753, parried 427, shield 1,235, skill 850, armour 1,282, no misses. Arena, the player's blows: landed 653, blocked 131, parried 119, shield 140, skill 128, armour 152.

Determinism: the direction scene twice, 18 of 18 fingerprints equal. Strict clippy clean, 44 tests pass. The fingerprints differ from main by design.

The frontal pace is the thing to judge in play. The two levers are `FL_HIT_RATE` (1.75: a blow at even points lands 35 in 100) and `FL_GUARD` (0.10 per skill point: the share of answers on time). `FL_HIT_RATE=3 FL_GUARD=0.07` would be a middle setting.

## The render cue, rejected (2026-10-13)

Gota's verdict on the code-posed cue below: it looks funny and unsophisticated, and the cue for a turned blow in particular has to look good or not exist. The commit (4c8dfa8) is parked on `backup/parry-render-cue` and the branch is reset to the sim core (9937470). A procedural pose on the skinned mesh, judged from stills, was the wrong tool; the cue waits for Astra's clips (handoff 115) and a decision on how a turned blow should read. The plumbing in that commit (the signal in the stagger channel, the jolt code in the hit-flash column) can be taken back from the backup when the clips exist.

### What the commit did

Gota's play: a turned blow looked like a bug, the attacker swinging through and the victim standing still. And nothing showed the answer itself. Both now have a pose, code only, until Astra's clips (handoff 115):

- The fourth anim2 float, the stagger until now, carries the cue on both render paths: a block as 1 plus how far the shield is up, a parry as 2 plus the same (from the defend state and its timer, `guard_amount`), a jolt as 3, 4 or 5 (shield, blade, armour) plus how much is left. The jolt comes from a code the damage apply writes into the hit-flash column (`JOLT_*`, values 16 and up, 9 ticks); the kernel counts it down, the red flash ignores it, and the GPU pack sends it whatever the hit-flash setting says. The impale check treats a jolt as no flash.
- `unit_pose.wgsl`: the skinned shield arm lifts forward and in (0.9 and 0.35 rad on the upper arm) with the elbow bent (0.9 rad) for a block; the sword arm goes up and across (1.3 and 0.6 rad, forearm 0.4) for a parry; an armour flinch rocks the body 0.12 rad through the stagger's turn. The parts path lifts the shield as the wall does and raises the blade 1.3 rad.
- Looked at: the arena's frontal lane from the east end of the line (`tmp/runs/parry/shots/arena-c_*`, the burst sheet `burst_sheet.png`): blocks show as shields raised square in front at chest and face height while the enemy swings; nothing detached. The parry and the jolts were too brief to catch in stills, so their look is unverified by eye.

## The clips in (2026-10-15)

Main (the occlusion culling merge, ab80162) merged into the branch as 73618cf. Then Astra's knight package (handoff 118): `assets/units/knight.glb` with the 20-joint skeleton (a `forearm_twist_l` between the forearm and the hand) and the corrected shield mount, the ten legacy clips migrated, and `knight.block.json` (0.80 s, guard 0.20, recoil 0.40, recover 0.50) and `knight.parry.json` (0.90 s, guard 0.15, deflect 0.20, recover 0.50).

- The sim's timelines now follow the clips: block raise 6, up 6, length 24 ticks; parry raise 5, up 3, length 27. The guard window ends at the block's recoil and the parry's deflect. Fingerprints change by design; the direction scene is deterministic (18 of 18 twice).
- The player: `SKEL_CLIP_SLOTS` 14 to 16, slots `block` and `parry`; the loader reads `events.guard` as the clip's key time. The fourth anim2 float carries a block as 1 plus how far the clip has run and a parry as 2 plus the same (both render paths); the pose shader plays the clip from the sim's timer as authored, legs keeping the gait on the move. The counter stab plays from its half wind-up as any stab does.
- Removed at Gota's word (2026-10-15, the commit amended): an ease in from ready over the clip's first 0.1 s and out over its last, and a blend from the parry's guard pose into the counter stab over the rest of the wind-up. Gota despised both motions next to Astra's clips. The clips cut in at their first sample and out at their last; the counter stab starts from the parry's guard pose without a blend.
- The man-at-arms shares the knight's clips but his model has the old 19-joint skeleton. He keeps copies of the old clips as `man_at_arms.<clip>.json`; a kind's own file wins over the shared one, and a shared optional clip with other joints is skipped. He has no block or parry until his model is migrated (handoff 119 for Astra).
- Looked at: the arena from the east end of the line, a 24-frame burst (`tmp/runs/parry/shots/clips_sheet.png`): the shield comes up square in front of the face, the blade rises for the parry, no broken limbs.
- Commit: the clip integration after 73618cf.

## The parry hold and the twitch (2026-10-15)

Gota's play: the parry into the counter was rushed or slow by luck. Cause: a parry that stopped the blow switched to the counter 0.17 s into its clip (the blade is timed to be up exactly at the strike, and the counter followed at once), while a late parry played all 0.9 s. Gota picked fix 1: the stopped blow marks the parry (`GUARD_STOPPED`, the charge bit, free in the defend state), the clip plays on to `PARRY_COUNTER_AT` (11 ticks, 0.37 s, through the deflect at 0.20 s), and the kernel starts the counter there. The late parry still plays through.

Gota's second ask: a blow the dice stopped (shield, skill or armour credit) showed nothing; a twitch that reflects the blow's direction. Built render only: the damage apply writes the side (1 front, 2 his left, 3 his right, 4 behind) and `TWITCH_TICKS` (9) into the hit-flash column as values 16 and up; the kernel counts the low nibble down and drops it; the red flash and the impale check ignore those values; the GPU pack sends them whatever the hit-flash setting says. The fourth anim2 float carries it as 3 plus the side plus how much is left. The pose: a frontal blow pitches him back 0.10 rad through the stagger's pitch, one from behind forward 0.08, a side blow 0.04 back plus the struck shoulder turned back 0.25 rad (the chest joint on the skinned path, the shoulder turn on the parts path), quick in over the first quarter, eased out by the end. Nothing new drawn.

Not looked at in motion by me; Gota judges in play. Strict clippy clean; the direction scene deterministic (18 of 18 twice); fingerprints change by design (the hold).

Also this day: Gota's callout on my explanation of the fight (one mechanism explained as two steps, the dice roll's credit split presented as three things); the rule is in `docs/internal/002` and the explain-like-gota memory.

## Open

- A man who answers while his cooldown has a long way to go comes back ready when the clip ends, which can be sooner than his cooldown would have been (the answer wait plus clip is 21 to 39 ticks, the cooldown 25 to 60). M2TW's reaction has the same property. If play shows defenders swinging faster than non-defenders, the fix is a leftover cooldown after the clip.
- The hit rate and `GUARD_PER_SKILL` are starting values; the fight length Gota wants sets them.
- The counter is a stab at half the wind-up and warns nobody.
- Gota, 2026-10-13, after the reference sheets: the `defend_*_slashrl` clips read as counterattacks rather than defences, and what they defend against is unclear (the data lists them as the answer to a right-to-left slash). The parry reference is `defend_mid_slashlr`, the block `defend_mid_stab`. Later, not soon: decide the outcome at the swing's start so the attacker can play the fail clips (plan 024, later list).
