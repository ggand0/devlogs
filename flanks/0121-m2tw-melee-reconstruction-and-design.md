# 0121 — M2TW melee, reconstructed, and a design for ours

Written by Claude Opus 5.5. Date: 2026-09-25. Research and proposal only, no code. Follows 0119 (rear-rank creep) and 0120 (animation and data evidence).

Gota asked: much of M2TW's unit-level formation and attack logic is still unknown; search the devlogs, research again, and make an educated guess of how it works, or design a better one.

## Where "only the front two ranks fight" came from

Not from the research. The record says:

- 0035: an agent built "only the front K ranks fight"; Gota rejected it as not M2TW.
- 0036: launch M2TW had that behavior, players hated it, patches walked it back. Default M2TW (guard off): everyone presses and tries to fight; guard mode: the first line fights.
- melee-brawl-band plan: Gota, 2026-07-18: "only the first two rows fight, complete garbage".
- 0042: the enemy rest distance went back to 1.4 m for weapon clipping, which physically restored the two-rank line. That is main today.
- The phrase most likely comes from M2TW's spear wall: "the front two rows fight simultaneously, with losses replaced by the back ranks" (Steam unit guide).

## M2TW, reconstructed

Tiers: [DATA] game files in the install; [EXE] class names in medieval2.exe; [EOP] reverse-engineered engine structs (EOP-Labs/M2TWEOP-library, types/unit.h and battle.h, read this session); [OBS] player observation; [GUESS] mine, with the reasoning.

### Unit level

1. An attack order is a unit task (`UNIT_TASK_MELEE_ATTACK`, `_FORMED`, `_PHALANX`, `ATTACK_ENGAGE`) [EXE]. The unit's move carries a formation id, a front-rank soldier count and a target angle (`targetPos`) [EOP]. The path is computed once and the formation stops updating 20 m before its last point (`formation_hold_distance 20.0`) [DATA]. [GUESS] The frame therefore stops where the approach ended, near the enemy's face, and does not chase the enemy's center of mass.
2. A charge is its own task (`CHARGE_FORMED`, `CHARGE_UNFORMED`); the unformed charge ends once 75% of the infantry have charged [DATA].
3. Engagement is counted per enemy unit: `engagedUnit { unit, engagedSoldiers, spearPoints, engagedRatio }` [EOP], queried by `IS_UNIT_ENGAGED_WITH_UNIT` [EXE]. [GUESS] The unit's melee state follows the enemies its men are actually fighting, not only the ordered target.
4. `UNIT_TASK_RESHUFFLE` and `UNIT_TASK_REFORM` exist [EXE]; losses are replaced by the back ranks [OBS, spear wall guide]. [GUESS] Reshuffle moves rear men up into the dead men's slots.
5. The AI's melee manager engages within 40 m (infantry) / 120 m (cavalry) and pursues up to 3x that [DATA].
6. Default depth is 3 to 5 ranks at 1.2 x 1.2 m [DATA, EDU]. Most men of an M2TW unit stand within a few meters of the front.

### Soldier level

7. Every soldier keeps his slot (`formationX/Y`), his displacement from it and an `isInFormation` bit all battle [EOP].
8. In melee he runs `ACT_ATTACK_MELEE` [EXE], whose state [EOP] holds: target unit and target soldier; an attack POSITION (`posX/posY`) and rotation; the chosen attack animation; a combo chain; recover ticks; `crowded`, `allowAttack`, `readyStance`, `isSideStepping`, `locked`; and the victim (`hintSoldier`, `hintAnim`, `hintStrike`) who plays the matching defence.
9. The attack animations fix reach and timing: each success clip records where the weapon lands (1.1 to 2.8 m ahead) and on which frame (0.75 to 1.3 s) [DATA]. [GUESS] The soldier picks an attack whose reach fits, walks to the attack position that puts the impact point on his target, then plays it. Success or fail is decided when the swing starts (success / fail clip pairs) [DATA].
10. Around each soldier the engine keeps up to 100 enemies and 100 friendlies with a per-soldier `collided` flag, and a `blockedCounter` [EOP]. The locomotion controller has a stage "stopping and waiting before move" [EXE]. [GUESS] A soldier whose way to his attack position is blocked by a friend sets `crowded`, cannot attack, stands in ready stance and counts; after a while he sidesteps to find a lane (`isSideStepping`), otherwise he keeps waiting. That produces "press up, wait, fill the gap" and the flow around the flanks, with no continuous pushing.
11. Every move is a discrete gait: short moves shuffle in four directions without turning (0.75 to 0.9 m/s); longer moves turn, walk and turn back, with turn animations covering 5-30°, 29-68° and 67-115° [DATA, EXE]. Nothing creeps.
12. Each victim counts his attackers (`targetCount`) [EOP]. [GUESS] Target choice avoids piling many attackers on one man.

Still unknown: the engage radius a soldier seeks targets within, the waiting time before a sidestep, how crowded is decided exactly, the target-choice weights, the hit roll. Only disassembly or live probes would pin them.

## What ours does differently (0119)

- Attack orders center the slot grid on the target's live center of mass. Slots sit inside the enemy and move every tick: the rear-rank creep and, when the ordered target is not the regiment in contact, the sideways slide (debug2 screenshot).
- Soldiers are steered by continuous forces (slot pull, 4 m surge, separation). There is no waiting state and no minimum pace, so blocked men lean on the man ahead and creep at 0.1 to 0.4 m/s.
- Gaps are never filled by rule while engaged (FL_RECTFIGHT off); rear men only drift into them.
- 500-man regiments are 15 ranks deep (M2TW: 3 to 5).

## Proposed design

Keeps what Gota likes about FL_RECTFIGHT off (pace, no flocking); every step maps to a numbered M2TW item above.

1. Contact frame (items 1, 3). When a regiment's men engage an enemy regiment (the count gate already in frontline.rs, but keyed to whichever enemy regiment they are actually fighting, not only the ordered target), its frame stops following any center of mass. It holds where its front rank met the enemy's face. It re-anchors only when the fight line itself moves by more than a stride (the enemy front collapses or retreats). Rear slots then sit behind their own front rank, so nobody is driven into a friend's back, and nothing slides the block sideways.
2. Soldier states instead of summed forces (items 8, 10, 11). Each soldier with no enemy in reach is in exactly one state:
   - in slot: stands, ready stance when the regiment is engaged;
   - stepping: walks to a point at a fixed gait (shuffle under about 1.5 m, facing kept; walk and turn above), then stops;
   - waiting: he wants to move but a friend's body blocks the way; he stands and counts ticks; after a wait he tries one sidestep toward a free lane, else keeps waiting.
   Separation stays as physics for shoves and overlap, but it no longer drives the walk cycle.
3. Seeking a fight (items 8, 9). A soldier with a reachable enemy within the engage radius (today's 4 m wide-acquire) and a clear path steps to an attack position (target minus his weapon reach), then swings as today. Blocked, he waits. That is who fights: whoever can physically get to an enemy, which stays about two ranks at 1.4 m but lets more men in wherever the line is broken or open.
4. Fill the gap (item 4). When a man in a front slot dies, the next living man behind him in the same file takes that slot and steps up one pitch (a forward shuffle). Local and immediate, no grid rebake, no mass movement.
5. Render. Shuffle left/right clips from Astra (0120 refs), walk forward/back for now; the clip is chosen from the step direction relative to facing.

Formation depth is the player's choice: a shallow formation that ends a fight fast is fine. Every step above must work for any files x ranks, any spacing (normal, loose, wall) and a single rank: the contact frame is placed from the regiment's own slot geometry (its front-most slot), "the man behind in the same file" is read from the slot grid (nobody behind, nothing moves up), and Blob regiments have no files, so they skip the fill.

Verification: the FL_DIAG_REAR numbers (target: idle rear men moving under 10% of ticks, and no creep under the shuffle pace), per-rank reach bins, casualty pace against main in the same 8 v 8, the sliding pair gone, then Gota's feel pass.

## Gota's decisions (2026-09-25)

- Approved; he wants to try it. Refactor of movement.rs at the end of the same branch, not before.
- The explicit crowding / waiting state (item 10, design step 2's "waiting") is deferred: we already have body collision. Caveat on record: collision stops the overlap, not the push, and 0119's creepers are exactly men pushing against it. The contact frame removes the main push (slots inside the enemy); what remains is the 4 m surge and gap filling. Build steps 1, 3 (without the wait), 4 and the fixed-pace stepping first, measure with FL_DIAG_REAR, and add the wait only if creep remains.
- Any formation and any number of ranks must work (see above); fast endings from shallow formations are fine.
- Astra gets the side shuffle clips: work/handoffs/HANDOFF-side-shuffle-2026-09-25.md.
