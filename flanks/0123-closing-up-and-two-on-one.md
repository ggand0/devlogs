# 0123 — Closing up behind the fighters; why a unit gives ground to two

Written by Claude Opus 5.5. Date: 2026-09-25. Branch `feat/melee-footwork`, continuing 0122. Commits c208f27 and cdc15d6.

## Gota's play test of e5214c6

"Much better than the previous version." Two things left:

1. A small gap still opens between the fighting front and the rear ranks over time. The rear men do react and move up, but it takes 5 to 7 seconds. Worst for spears (resources/unit_movements_debug4_092526.png, 5), smaller for the other melee kinds (debug6).
2. A unit seems to back up a little when two enemy units engage it at the same time. Maybe realistic, but why?

## 1. The gap: closing up (c208f27)

Cause: a front man who steps forward out of his slot to fight is alive, so the file close-up (which runs on deaths) never moves anyone up behind him. The ranks behind held their slots until the regiment's frame caught up, which it does only once the fighters' mean position has moved a whole rank forward, and then everyone walked up at 0.8 m/s. Spears fight from furthest out (reach 2.4 m), so their strip is deepest and the gap largest.

Fix: on a contact frame, a man behind the front rank keeps close behind the nearest file-mate ahead of him (same regiment, ahead along the facing, within half a file pitch sideways, seen in the neighbor scan). Once the gap passes half a rank he steps forward; once moving he closes to within 0.35 m of one pitch. Pace from M2TW's own clips: ready-stance `advance` 1.06 m/s for a small gap, `combat_jog` 2.87 m/s when the gap is more than a rank (read from pack.dat with tmp/scripts/m2tw_anim). A file-mate beyond the scan counts as a gap of more than a rank. Forward only, through open ground (a comrade in the 45° cone ahead stops him), and not while an enemy is in reach. A man in the front rank has no one to follow.

All-spear 4 v 4 (`FL_UNITS=2000 FL_REG_SIZE=500 FL_HEAVY_FRAC=0 FL_SPEAR_FRAC=1 FL_ARCHER_FRAC=0 FL_DIAG_REAR=1`), men per file in each rank-deep band behind the front:

| | band 1 | band 2 | bands 3+ |
|---|---|---|---|
| before | 0.58 to 0.71 | 0.51 to 0.64 | about 0.95 |
| after | 0.95 to 1.05 | 0.80 to 0.88 | 0.8 to 0.9, thinning toward the back |

The hollow behind the front is gone; the thinning collects at the back of the block. Pace unchanged (about 1,000 lost per side at 58 s either way). FORM, DIR and CHARGE hold their bands.

New effect, not fixed: rear rows now move on 70 to 95% of ticks at 0.3 to 0.45 m/s, and 44 to 78% of that is body contact. With the ranks closed up behind the fighters, the fight's shoves travel back through the packed file; the old hollow had insulated them. A standing soldier has no grip on the ground in our physics: any separation push, however small, turns into velocity (a 7 cm compression settles near 1.4 m/s). Candidate fix, for Gota to decide: a standing man plants his feet, so pushes under a threshold do not move him and bigger shoves still do (Coulomb friction on standing men). It is physics rather than a rule, but it changes every standing crowd.

## 2. Two on one (cdc15d6: `FL_PILE_N`, `FL_PILE_ATEASE`, edge logging)

`FL_TEST_PILE=1 FL_PILE_N=2`: two men-at-arms regiments attack one holding men-at-arms regiment of 500. The victim's front and rear edge (95th and 5th percentile of depth along its facing) against 1 attacker:

| | front edge at 24 s | rear edge | losses at 24 s |
|---|---|---|---|
| 1 attacker | gives 1.4 m (one rank) at impact, then holds | moves forward 1.5 m (files close up) | 82 |
| 2 attackers | falls back 5.6 m, steadily | pushed back 0.9 m, then closes up | 168 |

The victim's center is misleading on its own: it moves back as front-rank men die even if nobody steps. That is why the log now reports the edges.

It is not charge impact: both attackers had stopped charging before the victim began to give ground. It is not only the close-in step either: capping the fighters' step-in at advance pace (a temporary test, removed) left the retreat the same.

Why: separation is a spring between every pair of bodies closer than their rest distance, weighted by mass. Two regiments put more attackers against each victim front man, from the front and the corners, and each pushes. The victim's men answer only by stepping back toward their marks at the holding pace, so the summed push wins and the front gives ground about a rank every 6 seconds, while the attackers' contact frames follow the fight line forward. More bodies pushing moves a line back; M2TW documents mass pushing the same way ("units with big mass can push their enemies harder"). The standing-friction change above would make a braced line give ground only to a real imbalance, which would soften this too.

## Open

- Gota's call: standing friction (above).
- Sideways steps still play the forward walk until Astra's shuffle tables land.
- Not run yet: ROUT, SURROUND, JOIN, 200k perf.
- The movement.rs refactor, last on the branch. Fingerprint baselines need regenerating at the final behavior commit before it (the ones for e5214c6 are stale now).

## Gota's questions and the runaway-flank bug (after c208f27)

Clarified for Gota:

- Following: before, a rear man walked to his slot (his place in the grid), which moved only when a man ahead in his file died (the file close-up) or when the whole frame stepped forward. Now, in melee, he also follows the actual man ahead of him in his file.
- Who pushes whom: every pair of soldiers closer than 1.4 m between centers pushes apart like a spring, both ways. Fighters shoved back by the enemy (or stepping back) come closer than 1.4 m to the man behind; he is pushed back, closes on the man behind him, and the shove travels down the file. Rear men do not push the fighters forward: they step up only through open ground.
- Body size in the physics: two soldiers rest 1.4 m apart (a 0.7 m personal-space radius each), bodies are corrected positionally only below 0.9 m, and the model is about 0.6 m wide. M2TW's documented soldier radius is 0.4 m (0.8 m between centers). 1.4 m was restored for enemies in devlog 0042 because closer contact clipped weapons. Gota wants a way to see this later: a debug overlay drawing each soldier's circles.
- Plant your feet: today any push becomes velocity at once (force, acceleration, velocity, no ground friction), so a 7 cm squeeze makes a standing man slide. The proposal is static friction for a man who is standing (not stepping): the push has to exceed a threshold before he moves. Below it he holds his ground; true overlap is still corrected. Above it he is shoved as now.

Bug (resources/unit_movements_debug7_092526.png to 9): soldiers with no enemy in front of them keep walking forward, worst when a stretched line attacks a narrower enemy; its flank files walk far past the fight.

Cause: the contact frame moves the whole regiment forward, a rank at a time, whenever the fighters' mean position moves a rank into the enemy. The fighting center keeps pushing into the enemy block as its front dies, so the frame keeps stepping and drags every slot with it, flank files included. The close-up rule adds to it: a man who sees no file-mate within the scan (about 2 m) treats it as a big gap and jogs forward.

Proposed fix (not built):
1. Freeze the frame at contact, laterally and in depth. A file with no enemy in front of it holds its slots.
2. Replace the scan-based file-mate with an exact per-file lookahead: each tick the prep records, per regiment and file, where that file's front man actually stands and which slot he holds. A man k slots behind the file's front slot targets k ranks behind that man, forward only. Fighting files follow their fighters however far ahead they are, and a flank file whose front man stands still does not move. No "unseen mate means a gap" guess.

## Gota's ruling on the fix (same day)

Fix 2 (the per-file lookahead) rejected as cheating and a superficial hack: a flank soldier cannot know where his file's front man stands; a file that stays closed at zero gap is robotic; "forward only" reads as ignoring physics. The committed close-up rule (c208f27) has the same flaw and goes too. He wants what a real medieval unit in ranks does, or what M2TW does. M2TW has no frame moving through a melee (formation_hold_distance; slots kept) and no follow-the-man-ahead rule; its rear men seek fights themselves, wait when crowded and sidestep for room.

Proposed replacement (awaiting Gota's go): freeze the frame at contact; each soldier acts only on what he perceives: seek an enemy within a sight radius (tuned by play) through open ground, wait behind a comrade, sidestep left or right toward an open lane after a wait, hold his slot when no enemy is in sight; per-soldier reaction delay and stopping variation; standing friction so a shoved man braces (runners and charges unchanged). Collision is a 2D circle on the ground (rest 1.4 m between centers, 1.05 for wall pairs, hard below 0.9 m); Gota wants a debug overlay for it after the bug fixes.

## Seek / wait / sidestep / brace (796ff14)

Built per the proposal above. The frame is set once at contact and never moves. The file-follow rule is gone. Men of an engaged regiment look for an enemy within sight (FL_SEEK_R, default 15 m) and go to him through open ground: combat jog 2.87 m/s from beyond 3 m over open ground, advance 1.06 m/s for the last meters or with a comrade close ahead. Each man stops at 1.2 to 1.6 m of his own. A man blocked by a comrade waits, and sidesteps toward an open side in a share of one-second windows (FL_SIDESTEP, default 0.2). A standing man's separation push is reduced by 6 m/s^2 of grip (about a 10 cm squeeze).

Measured:
- All-spear 4 v 4 at 8 m sight: the hollow moved to the edge of sight (band 4 at 0.24 to 0.32). At 15 m and 30 m: bands behind the front 1.1 to 1.3 men per file (the block presses a little denser), thinning at the back. 15 m chosen.
- Rear idle motion in the same run: 67% of ticks with sidestep off, 76% at 0.35. It is mostly the press, men stepping into space freed by deaths.
- 200k, same build, FL_LOG_STEP: mean step about 8.5 ms at 4 m sight, 9.2 ms at 15 m (+8%; the load average of 7.6 to 12.4 was the game itself, Gota was running nothing heavy).
- Wide line (FL_PILE_FILES=90, one attacker against a 34-file block): nobody passes the far side of the enemy. The old moving frame alone did not reproduce the runaway in this test (the victim holds, so the fight line never advanced); the likelier source was the follow rule's "unseen file-mate means a gap" guess, now gone.
- Two on one: the victim's front gives 2.3 m by 24 s (5.6 m before the grip).
- FORM OK (slot err 0.62 to 0.89, wall 1.02 < normal 1.30 < loose 1.86), DIR rear 49.1 vs front 22.0 per hit, lone victim 1 of 500, CHARGE wall 342 v 308, open 288 v 383.

## Wide line against a narrow enemy (resources/unit_movements_debug10_092526.png)

Gota: a blue line much wider than the orange block engages at one end; once the center men engage, the right flank shifts right as if aligning, then stands there doing nothing, out of the fight.

Causes:
1. The shift: during the approach an attack order lays the slots around the target's center, so the line is walking sideways to center itself on the enemy. At contact the frame snaps laterally to where the men stand on average, so the flank men, still converging, find their slots back out to the side and walk out to them.
2. Standing idle: nobody on the far flank sees an enemy within the 15 m sight radius, so they hold their slots. The unit was not in hold (in hold nobody seeks at all).

M2TW: each soldier's melee action carries a target unit as well as a target soldier (EOP actAttackMelee: targetUnit, targetSoldier), and in the default mode every soldier of an engaged unit tries to fight; the surplus flow around the enemy's flanks (the wrap). Standing idle 30 m from a fight is guard-mode behavior.

Proposed (not built):
1. At contact keep the frame laterally where the slots already were (around the target's center at that moment) instead of snapping to the men's average, so nobody walks sideways at contact.
2. A man of a regiment attacking a target, with no enemy in sight, heads for that target regiment itself (he can see the enemy block and he has his orders), through open ground at the jog, waiting behind comrades and sidestepping as usual. Within sight he picks a soldier as now. Flank men then close on the enemy's flank and wrap around it, as M2TW units do. Hold (guard) regiments keep their slots.

## M2TW reference: a deep unit hits a wide line (Gota's test, 2026-09-25)

Gota played M2TW on this box: his deep unit (orange) attacked a wide, shallow French unit (blue). Screenshots: resources/footwork_debug_092526/m2tw_exp_0.png to 5 (all earlier feedback images moved to the same folder).

- Contact: the deep block drives into the thin line's center; the blue men beside the contact curl along the orange block's sides at once, bending the line into a V.
- Next seconds: the line becomes a U. Men nearest the fight peel off first and run to the orange flanks; the far ends remain lines, angled inward. Gota saw the flank men fidget in place for the first few seconds, then more and more ran in.
- Then the far ends dissolve into the arc (a few stragglers out wide), and finally a full ring: blue surrounds orange on every side, back included. No blue formation remains.
- The blue unit was the defender, so it is not about who holds the attack order: any engaged unit out of guard mode fights with all its men.
- The joining rolls outward from the contact: near men first, far men seconds later, far men running about 30 m. No sign of a hard leave-formation distance; clearly a delay.
- The orange block's formation turns into a blob; its side and rear men face outward and fight.

Refined proposal (awaiting Gota): (1) no lateral snap at contact; (2) for any engaged regiment not in hold, attacker or defender: a man with an enemy in sight goes to him at once; the rest head for the enemy unit they are engaged with (the ordered target if fighting it, else the nearest engaged enemy regiment), each after his own delay of 1 to 6 s from when his regiment engaged, so the line rolls up outward; open-ground jog, waiting, sidesteps and fighting whoever they meet as now; hold regiments keep their slots. Plus a wide-line scenario to measure the roll-up.

## Joining and the roll-up (372b115, aa41742)

Built per the refined proposal, then corrected twice after Gota watched it:

- 372b115: melee clock and fight point per regiment (frontline.rs); men out of sight of an enemy head for the fight point after a random 1 to 6 s delay; the frame stays laterally where the slots were (around the target's center) at contact; comrades already walking the same way no longer block. Gota saw front-row flank men loop between the fight and their slots: the seek was added on top of the slot pull, so whenever the seek dropped out (blocked, crowd, enemy out of sight) the slot pull walked them back. Fixed in the same commit: a man committed to the fight (an enemy he goes for, or joining) drops the slot pull until the fight ends; blocked, he stands.
- Gota then saw the wide line surround the narrow attacker within about 10 s, not staggered (screens resources/footwork_debug_092526/debug9 to 11). A random delay from the engagement start fires the whole line at once. aa41742: a man joins when he sees a comrade of his own regiment close by running toward the fight, reacting per half-second window with chance 0.35 (FL_JOIN_REACT), and keeps going once on his way (from his own velocity). The wave starts with the men who see the enemy beside the contact and travels outward.

Measured with FL_TEST_PILE=1 FL_PILE_N=1 FL_PILE_FILES=12 FL_PILE_ATEASE=1 FL_PILE_VICTIM_FILES=100 and a locked camera (FL_CAM_X=0 FL_CAM_Z=40 FL_CAM_DIST=120 FL_CAM_PITCH=1.05 FL_CAM_LOCK=1, tmp/scripts/clip.sh): the center wraps within seconds, part of the far flank is still in line 23 s after contact, and men jogging in peak at 72 to 80 at the start and taper over 30 s. At 120 m the soldiers are still small in the frame; a closer FL_CAM_DIST would help the next review.

Battery at aa41742: FORM OK (slot err 0.62 to 0.89, wall 1.02 < normal 1.30 < loose 1.86); DIR per hit front 20.5 / side 26.6 / rear 48.7, lone victim 0 of 500, yaw dev 0.00; CHARGE wall lane 332 v 275, open lane 268 v 368.

## Far flanks (54f38ca)

Gota: the center-leaning men joined in a staggered way, but the far flanks never moved (resources/footwork_debug_092526/debug12.png, both flank blocks still dressed after 200 losses). The wave died: a comrade going was noticed only inside the 2 m neighbor scan, which a jogger leaves in under a second, and once the men near the contact had gone the flank block's inner edge saw nobody going at all.

Fix: a comrade running to the fight is noticed within 8 m (JOIN_SEE_R, on the every-8th-tick acquisition scan), and each man has a patience of 8 to 25 s (FL_JOIN_PATIENCE) after which he joins because the fight is on, sight or not. Recorded with the locked camera: the center wraps in seconds, the flanks converge from about 20 s, all in by about 40 s; jogging joiners come in two waves (40 to 60 early, about 150 when patience runs out). DIR: rear 48.5 vs front 20.6 per hit, lone 0 of 500; CHARGE wall 333 v 271, open 271 v 365.

## Run-back fix and verification (5a5f8ff)

Gota (debug13.png): some joiners came toward the center enemy, ran back to the flank rows, and went out again. Cause: joining was inferred every tick from the man's speed toward the fight; a joiner who slowed (blocked, crowd, sidestep) stopped counting as joining, and his slot pull, a run at 15 m, took him back. Fix: a per-soldier out_form column (M2TW's isInFormation): set when he fights, reacts to an enemy in sight or a comrade seen going (chance per half-second window), or runs out of patience; cleared when his regiment's melee ends, when a regiment with no attack order re-forms where it stands. Out of formation he never dresses on his slot. Roll-up retuned from the logs: comrade seen going within 6 m, reaction 0.2 per half second, patience 25 to 50 s.

Verified (logs first, then stills; Gota played 20k and 200k and approved the look):
- Wide line vs deep attacker: nobody runs back from the fight in any second of a minute; men out of formation 0, 131, 253, 316, 358, 375, 392, 400 over the 20 s after contact.
- 8 v 8, about 2 minutes: moving away from their regiment's fight, 50 man-samples out of formation and chasing an enemy behind them, 17 out of formation otherwise, 2 in formation once.
- FORM, DIR (rear 48.9 vs front 20.9 per hit), CHARGE (wall 340 v 283, open 275 v 374), SURROUND (pocket gone at 68 s, line 88), PILE 6-on-1 (5 of 6 engaged, victim 500 to 28 by 69 s): all hold.
- ROUT and JOIN compared against main built from `git archive main` (tmp/notes/rear/mainsrc, target tmp/notes/rear/maintarget, test-only FL_AUTOSTART and FL_LOG_STEP added there): ROUT does not break within 72 s on either (main 60 of 1000 alive, branch 156, slower pace). JOIN's B stalls behind ally A on both (10 to 17 m short): ally routing exists only under FL_RECTFIGHT. Not regressions.
- Determinism: DIR run twice, 19/19 equal. Baselines recorded: tmp/hash-baselines/{dir,arch}-footwork-5a5f8ff.
- Pipelined vs inline (FL_PIPELINE=0): the branch differs from the first sample. Main differs too (11 of 19 samples equal), so the equivalence claimed in devlog 0079 was already broken on main; not investigated. hashcmp.sh sorts ticks as text, so its "first mismatch" is lexicographic.
- 200k sim cost (FL_LOG_STEP; the load average of 7 to 9 was the game itself, nothing else heavy ran): main step 6.5 ms / grid 3.8 ms / 351 spike lines; branch step 9.6 / 3.9 / 804; branch with FL_SEEK_R=4 step 9.2. The sight radius is not the cost. The extra ~2.7 ms is per-neighbor work added to the fused scan (cone tests, random reads of vel_snap and group for the comrade-walking-away and comrade-going checks) plus the per-tick velocity snapshot. Gota sees fps capped below 120 and dips to 60 to 70 at 200k: perf work for the next session.
