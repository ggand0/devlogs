Written by Claude Fable 5.1

# 0131: Footwork v2, how a soldier thinks (the reference, at dc41a22)

**Update, 5639239 (devlog 0132):** "the enemy crowd" in steps 4 and 6 below is now read from one field for everyone (the 8 m density grid: crowd cells with at least 4 enemy men, each with its men's centroid and box; a man heads for the nearest point of the nearest crowd cell's box, and for the centroid once inside), not from a per-regiment mass map. And step 2 has a phase before the melee for a charging regiment: the crash, during which its frame keeps moving, the charge pace stays on and nobody leaves formation, until the enemy has stopped the block or its rear has arrived.

Branch `feat/melee-footwork`, commit dc41a22. The plain-words description of the footwork as it runs now, for anyone (including a later session) who needs the logic without reading movement.rs. Pictures: work/notes/vis/016-footwork-flow.png (the decision flow, one soldier, one tick), work/notes/vis/011-touch-wave.png (what he perceives and how the wave travels), work/notes/vis/013-mass-map.png (the enemy crowd as nine bins). Design history: 0121 (the M2TW reconstruction), 0123 (the first build), 0129 and 0130 (v2 and its feel checks). The two fixes proposed in 0130 (the crowd's boxes and the crash phase) are not in yet; where they change this description it is marked.

## The idea in one paragraph

A soldier knows three things and nothing else: what is inside arm's reach around him (the 2 m box he already scans every tick for bodies), what the comrades right next to him just did (a start event reaches him when one of them, within 6 m, begins to fight or sets off for the fight), and where the enemy crowd is (his regiment's enemy, binned 3 x 3 each tick: nine spots he can face and walk toward, and "in sight" when one is within 15 m). Everything he does follows from those three, plus two clocks his regiment keeps: whether it is in melee at all, and how long, for his patience. Nothing scans the field for him.

## One tick, one man (the picture work/notes/vis/016-footwork-flow.png)

1. **Touch.** He scans his 2 m box: who pushes him, is an enemy within his weapon's reach, and who is the nearest enemy in the box at all (up to about 3 m). This scan is main's; v2 only reads one more thing out of it.
2. **Is my regiment in melee?** Its clock runs once 3% of its men (at least 4) had an enemy in reach at the same time, and stops when no enemy is engaged with it any more. Out of melee he dresses on his slot with the line's facing and this is the end of his tick.
3. **An enemy in reach?** He fights: wind-up, strike, recover, the last metre closed, his eyes on the man. Main's swing rules, untouched. His first swing of the melee counts as a start for the comrades around him.
4. **Still in formation?** He keeps his slot and the line's facing (M2TW's formed man does not turn on his own). But he is deciding whether to leave: every half-second window he rolls a 20% chance while any of these holds: an enemy stands in his box; the enemy crowd is within 15 m (he looks every 8 ticks); he remembers a comrade next to him setting off or starting to fight (the memory lasts 2 s and is refreshed while that comrade is still going within 6 m). If none of that ever happens, his patience, 25 to 50 s of his own, sends him anyway. Leaving is for the whole melee: the slot no longer pulls him.
5. **Out of formation, an enemy in my box?** He goes for that man: the advance pace, stops at his own distance (1.2 to 1.6 m), faces him. A comrade in the way: he waits, and in one one-second window in five he sidesteps toward the open side.
6. **Out of formation, nobody in my box?** He heads for the enemy crowd: the nearest of the nine bins of the regiment his own fights. (After the 0130 fix: the nearest point of the crowd's boxes, also of the two nearest other enemy regiments, refreshed every 8 ticks.) He jogs over open ground (2.87 m/s), walks behind a comrade (1.06 m/s), waits and sidesteps when blocked, and faces where he is going. The moment he sets off (0.5 m/s) the comrades within 6 m of him get the memory of step 4, again every half second while he keeps going.
7. **Melee over.** The flag clears, a regiment without an attack order re-forms where it stands, an attacker's order lays its slots again, and he walks back to his slot.

What the regiment does for him, once per tick: keeps the melee clock, picks the fight target (the ordered enemy if alive, else the nearest formed enemy), bins that enemy's living men into the mass map, holds the contact frame (its slots stop following the enemy's centre once its front met the enemy's face), and advances patience's clock.

## Scenario A: the second rank at the point of contact

The blocks meet. The front rank has enemies in reach and fights (step 3). The man behind him, 1.4 m back, has the enemy 2.6 m ahead: inside his box, not in his reach. He is still in formation, so he rolls to leave (step 4) with an enemy in sight, and the front man's first swing gives him the memory too; within a second or two he is out. He walks up (step 5), meets the front man's back, waits, faces the enemy, sidesteps now and then. When the man in front falls or steps aside he closes the last metre and fights; his first swing is the cue for the rank behind him. Rank by rank the block commits, in about two seconds for four ranks, and the men wait their turn pressed behind the front, facing the fight.

## Scenario B: the far flank, 30 m from any enemy

His box is empty, the nearest crowd bin is 30 m away, nobody near him has moved: he stands in the line, facing forward, and rolls nothing. The wave comes down the line toward him: the man on his inner side commits and sets off, and that start reaches him (within 6 m). He now rolls 20% per half second while the memory holds, refreshed as his neighbor keeps going, so he usually follows within two or three seconds; if the neighbor is out of his 6 m before the roll lands and nobody else sets off near him, patience takes him 25 to 50 s after his regiment's clock started, never not. Once out he faces the crowd and jogs for its nearest bin, walks when a comrade is ahead, waits and sidesteps when blocked. Arriving, his box fills with enemies, he closes to his own distance and fights. On the way back, when the melee is over, he returns to his slot with everyone else.

## What this replaced, and why (0129, 0130)

bee555e polled: every idle man of a fighting regiment read every man within 15 m every quarter second and every man within 6 m for runners, and validated a remembered enemy with random reads every tick. That was ~380 cycles per soldier over main at 200k and grew with sight and density. v2 costs one compare per enemy candidate in the scan everyone runs anyway, one small query per start event (rare), and nine distance checks for a man who needs the crowd. The behavior it keeps is Gota's approved one: the roll-up from the contact outward as a wave, the wrap of a wide line around a narrow block, and patience as the backstop; the behavior it changed is that a man 4 to 15 m from an enemy with nobody near him no longer spots that enemy directly (the crowd within 15 m now covers this), and that men out of formation face and head for the enemy crowd rather than a centroid.

## Knobs

FL_FAR_LOOK=1 (the bee555e perception in the same binary), FL_SEEK_R (the crowd-in-sight distance, 15 m), FL_JOIN_REACT (0.2 per half second), FL_GO_MEMORY (60 ticks), FL_JOIN_PATIENCE (the upper end, 50 s), SET_OFF_SPEED (0.5 m/s, a constant).
