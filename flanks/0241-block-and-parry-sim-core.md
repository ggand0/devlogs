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

## The counter spread and the exchange (2026-10-17)

Gota, after seeing the twitch (FL_TWITCH, 7f69eaa): every parry showing the same 0.37 s before the counter is boring; scale it by skill with a spread, let the counter be answered with diminishing odds so a chain can be seen, and never for ever.

- 886401b: from the tick after his blade stops the blow, the man launches the counter with a chance each tick of `COUNTER_PER_TICK` (0.04) per point of defence skill, at `PARRY_COUNTER_AT` (0.37 s) for sure. A knight (0.24 a tick) mostly counters at 0.20 to 0.30 s, one in five at the cap; a spearman about half at the cap. Whether he counters at all is a roll at the stop, `COUNTER_CHANCE_BASE` 0.8 plus 0.03 per skill point less fatigue; a parry without a counter plays its whole clip. A per-tick chance instead of a drawn time because the defend byte has no room to store a drawn tick while the timer drives the clip. FL_COUNTER_TICK, FL_COUNTER_BASE, FL_COUNTER_SKILL.
- 468aec0: the counter warns its target like any swing, one deeper in the exchange. A new per-soldier column `chain` holds the depth of the blow a man is answering; the warning carries `chain`. The parried man may answer the counter from the bind anywhere in his recovery (the 0.4 s rule is skipped when the warning is a counter from the man he struck). His on-time share is multiplied by `CHAIN_SHARE` [1.0, 0.6, 0.3, 0.1] by depth and is nothing beyond, so no exchange runs on. The counter's wind-up rounds up (a man-at-arms' 4.5 ticks to 5) so it can be parried on time by a raise of 5; a block (raise 6) is on time only against a knight's counter. FL_CHAIN as a comma list.
- Counts: the dir-test and arena lines print parries by depth. Direction scene at 44 s: [724, 67, 3, 0, 0]; the arena at 36 s: [241, 31, 2, 0, 0]. Deterministic (the 44 s stat line equal twice).
- Gota, after play: the counter base stays 0.8, and a counter is never blocked (the parry earned it): the answer to a warning with depth above 0 is always the parry. Arena at 36 s after it: parries by depth [244, 78, 15, 3, 0], a third counter parried a few times a match, a fourth never seen.
- Not looked at in motion by me; Gota judges in play. The parry of a counter starts from the stopped cut's pose with no blend, like the counter's own cut.

## Astra's 19-joint knight and the spearman's parry in (2026-10-17)

Commit 2b5f669: handoff 123's knight package (`assets_dev/knight/defend_19_v1/compatible`, model on the 19-joint family skeleton, block and parry re-exported, ten legacy clips as before) and handoff 125's spearman package (`assets_dev/spearman/parry_v1/compatible`, corrected shield mount, shield-led parry on the knight's parry timeline). Hashes checked against both handoffs. The loader lines: the knight 19 joints with block and parry; the man-at-arms now plays the knight's block and parry (his ten clip copies are byte-equal to the knight's legacy files and can go, with Gota's word); the spearman loads the parry, no block. Astra's warning stands: the counter out of the spearman's parry cuts to the stab's half wind-up with the spear sideways, about 2.2 m of spear displacement, until a counter-specific entry exists.

## The man-at-arms' left arm (2026-10-17)

Gota in play: the man-at-arms' left arm is broken in every attack, block and parry. Not the clips (byte-equal to the knight's, which is fine) and not the player: in `man_at_arms.glb` L0, 87 sleeve vertices of the left arm (33 on the upper arm, 54 on the forearm, within 9 cm of the arm axis) are skinned to the `shield` joint at weight 1.0 with the shield part id, so every hand turn carries the sleeve. The knight, the spearman and the man-at-arms' L1 have none. Present since the model (9971fe7); the block and parry made it obvious. Handoff 128 asks Astra to reweight them.

Fixed by Astra (handoff 132, assets_dev 78f7353), in the game as 1a101aa: the sleeve weighted to the arm, and the shield board 100 mm out and 30 mm down so the arm clears it through all twelve knight clips. The count reads 0 on the upper arm and 10 on the forearm, all at the 8 cm edge of the test next to the hand, which is the strap anchor, not the sleeve. Gota noted the right arm looks slightly odd in the shared stab; left for later, no cause diagnosed.

## The twitch on the spearman and the man-at-arms (2026-10-17)

Gota: spearmen show no twitch. Cause: a spearman answers nearly every blow he sees (the answer is set for late ones too), the sim wrote no twitch for a defending man, and his block has no clip, so a blow the dice stopped on him showed nothing at all. Fix (one commit after 2b5f669): the sim writes the twitch for a defending man as well; the build shader shows the answer clip only when the kind has it (a per-kind bit mask in `params.parry.w`, block bit 2k, parry bit 2k + 1, filled from the skeleton tables) and the twitch otherwise. The knight and the man-at-arms keep their clips; the spearman twitches on a block and plays his parry; the parts path shows the twitch for any answer.

## The spearman's block and his counter (2026-10-17)

Gota: reuse the knight's block for the spearman, and he does not see the counter out of a spearman's parry. Commit a905f33 copies `knight.block.json` in as `spearman.block.json`; both models are on the 19 joints, the loader takes it (the charge scene's log lists the spearman with block and parry). The right arm holds the spear where the knight's hangs the sword; Gota judges it at game zoom, Astra's spear arm replaces the file when it comes.

The counter: measured with temporary prints in the charge scene (spearmen against knights, 50 s), not committed. The sim launches it on spearmen as on knights: knights 446 counters of 454 stopped parries, spearmen 108 of 121. What differs is the share of parries that stop the blow at all, by skill (0.1 per point): knights 433 of 769 parry answers on time, spearmen 130 of 658. So 4 spearman parries in 5 play the whole clip with no counter, and the rest show a 0.17 s half-poke, since his stab's contact is at 0.33 s and the counter starts at half the wind-up. Commit c5fcf33 reads the share from `FL_COUNTER_WINDUP` (default 0.5) so 1, the full wind-up, can be tried. Also seen: spearmen answer knights' counters (depth 1) more often than they parry a plain blow, 402 against 228, since a counter is always answered with the blade.

## The counter window by kind (2026-10-17)

Gota, with the full stab as the counter (bfca48d): the spearman's parry is shield-led, so his counter should wait for the deflect to play. `PARRY_COUNTER_AT` became `COUNTER_WINDOW` per kind, (from, by) in ticks: sword kinds (0, 11) as before, spearman (11, 15), 0.37 s to the recover event at 0.50 s. Measured in the charge scene (45 s, temporary print): knight launches at ticks 5 to 11 as before; spearman at 11 to 15, 60 percent at 15. FL_PARRY_SHARE (bfca48d) sets a kind's parry share from the command line for a look.

## The 200k benchmark against main (2026-10-17)

The perf rule: the AI battle (`FL_TEST_FRONT=1 FL_UNITS=100000`), 2560x1360, camera locked, 120 s, the mean of the last 60 s, main ab80162 built from `git archive` into `target-agent` (`tmp/runs/perf4/bin/flanks-mainab`, md5 3869e86c), the branch at 5afe53a (`flanks-parry`, md5 8645ab1b). Order main, branch, branch, main in the low flank view, then main, branch in its mirror. Load average 4 to 12 (the game's own threads; docker idle), GPU clock 1830 to 1905 MHz a minute in. Script `work/scripts/perf/occl-bench.sh`, logs `tmp/runs/perf4/logs/p-*.log`, summary `occl_summary.py`.

| View | main fps (ms) | branch fps (ms) | Frame | Unit draw | Vertex runs | Alive at 120 s |
|---|---|---|---|---|---|---|
| Low flank, run 1 / 2 | 208.7 (4.80) / 206.2 (4.86) | 208.0 (4.81) / 207.3 (4.83) | 0.00 ms | 2.41 to 2.21 ms | 33.7 to 31.4 M | 178k to 195k |
| Low flank mirror | 161.4 (6.20) | 164.3 (6.09) | -0.11 ms | 4.42 to 4.27 ms | 62.4 to 60.6 M | 178k to 195k |

Equal within noise. The unit draw is 0.15 to 0.2 ms cheaper on the branch with 17k more men alive, since the slower fight leaves 17k fewer fallen on the ground in view; the pose, build and shadow passes are the same. No error lines, no run pinned at 60 fps.

The tick, from the last fps sample of the first flank runs: inside the fixed tick on the main thread p50 1.5 ms on main, 1.4 on the branch; the kernel's step on the job threads 9.2 to 10.5 ms with 17k more men alive and 137 events a tick against 63 (every wind-up now sends a warning); frame with a tick p50 5.5 ms on both.

## The spearman's block in (2026-10-17)

Gota: the knight's block on the spearman is bad, the grip jumps to hold the spear upright. Astra's own spearman block (handoff 136, `assets_dev/spearman/block_v2/compatible`, SHA-256 0c06db5e, after the defend_stab_mid reference with a shallower recovery) replaces the copy; the same 0.80 s timeline, the model unchanged (09026621). Loader lines from the charge scene: the spearman has block and parry. Handoff 122's three items are all delivered.

## Open

- Gota, 2026-10-17: a second parry motion, so not every parry is the same clip, and an attack clip that serves both the regular attack and the counter, so the counter stops cutting into the stab's half wind-up. Both are clips for Astra, after the spearman's block.

- A man who answers while his cooldown has a long way to go comes back ready when the clip ends, which can be sooner than his cooldown would have been (the answer wait plus clip is 21 to 39 ticks, the cooldown 25 to 60). M2TW's reaction has the same property. If play shows defenders swinging faster than non-defenders, the fix is a leftover cooldown after the clip.
- The hit rate and `GUARD_PER_SKILL` are starting values; the fight length Gota wants sets them.
- The counter is a stab at half the wind-up; it warns its target since 468aec0.
- Gota, 2026-10-13, after the reference sheets: the `defend_*_slashrl` clips read as counterattacks rather than defences, and what they defend against is unclear (the data lists them as the answer to a right-to-left slash). The parry reference is `defend_mid_slashlr`, the block `defend_mid_stab`. Later, not soon: decide the outcome at the swing's start so the attacker can play the fail clips (plan 024, later list).
