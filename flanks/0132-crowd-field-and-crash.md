Written by Claude Fable 5.1

# 0132: The crowd field and the crash of a charge

Branch `feat/melee-footwork`, commit **5639239** on dc41a22. Built while Gota was away, on his go for "the field approach if it is cheap and fixes the issue", after the 5 v 5 feel check of 0130 (three-part blobs, no charge impact). Picture: work/notes/vis/017-crowd-field.png. The soldier's flow in 0131 still holds; steps 2, 4 and 6 there now read as below.

## The idea, plainly

A soldier needs to know where the enemy crowd is when nobody is within his reach: to face it, to walk toward it, to notice it is close. Every scheme so far gave him a stand-in for that: a fight point (one centre), a face rectangle (a footprint that stuck out past the men), a mass map (nine points per regiment, so the ranks behind converged on three spots and each pair curled into a blob). Each was a per-regiment heuristic and each broke on the next scenario.

The game already computes the thing itself every tick: the 8 m density grid that the front line is drawn from. So the crowd is now a field, one for everyone:

- A cell with at least 4 men of a team actually standing in it is a **crowd cell** of that team. It keeps their centroid and the box they stand in.
- Two raster sweeps give every cell the nearest crowd cell of each team (the same trick as a distance transform, ~4k cells, tens of microseconds).
- A man reads the cell he stands in: the nearest enemy crowd cell, and heads for the nearest point of the box of men in it, the crowd's real edge; once inside that box, for the men's centroid. A crowd cell within 15 m (FL_SEEK_R) is an enemy in sight for a man still in formation.

Why it behaves: men in front of an enemy line all aim straight ahead and spread along its edge (the box is continuous, not three points); a wing man beyond the enemy's width aims at the nearest edge, so a wide line wraps a narrow block and only there; two regiments side by side are just more crowd cells, so a line battle stays a line. Nothing is per regiment any more: no fight target for heading, no neighbor lists, no bins. The fight point (the target regiment's centroid) remains only for the melee clock's bookkeeping.

## The crash of a charge, plainly

Before: a charging regiment's charge flag turned off the moment it engaged, the contact frame froze at first contact, and the wave dropped the slot pull of the ranks behind within two seconds. Only the front rank ever arrived at speed; a unit stopped the instant it touched the enemy. M2TW runs the charge task until most of the unit has charged (finish proportion 0.75, devlog 0120).

Now a charging regiment that engages is **crashing**: its frame keeps moving, the charge pace stays on, the slot pull keeps the ranks coming in, the front fights, and the melee clock waits (nobody leaves formation, no wave). The crash ends when the enemy has stopped the block, read off its centroid's smoothed forward speed (under 0.3 m/s, after at least half a second), or when its rear has had time to arrive: the block's depth over the charge pace (4 m/s), capped at 8 s. Then the frame freezes where the fight line is, the melee clock starts, and the footwork takes over. A defender's melee starts at contact as before; an attacker walking in (not charging) stalls at once. Everything is regiment state the frame already used, plus one smoothed speed.

Two versions were measured on the way and rejected: a flat 8 s cap (every crash ran the full 8 s; the two-on-one victim died twice as fast as on bee555e) and a stall on the fight line's speed (the front stops the moment it fights, so every crash ended in half a second and the rear never arrived).

## Details

frontline.rs: `InfluenceField` gains per team the unblurred `count`, `pos_sum`, per-cell `crowd_box` (min and max of the men's positions, from per-chunk scratch in the same parallel splat), the `crowd` map (nearest crowd cell index per cell, u32::MAX where a team has no crowd anywhere) and `crowd_c` (centroids); `rebuild_crowd` runs after the density blur; `crowd_snapshot` hands the maps to the tick job as `CrowdMaps`. MASS_N = 4 men. The splat uses the floor convention for cells; the crowd reader uses the same. The module doc now says what the field carries for soldiers (still not the front line: units never steer by the contour).

movement.rs: the job copies the maps at prep (`field_*`, about 100 KB); `crowd_goal` gives the goal and its distance squared; the kernel's `mass_goal` is computed from it exactly as the mass map was (every tick for a man out of formation with nobody in his box, on look ticks for a man in formation, SIGHT_MASS bit for the reaction). The charge boost column is `charging || crashing`. GroupData: `crashing`, `crash_ticks`, `crash_cap`, `adv_speed`; the mass map and slot extents are gone.

frontline.rs update_groups: `adv_speed` is the EMA (0.1 per tick) of the centroid's forward displacement over the fixed tick; the crash starts when a regiment that was charging becomes engaged with its melee clock at 0, ends on stall, cap or disengagement, logs "CRASHES" / "CRASH ENDS (n ticks)". The melee clock and the contact frame both wait for `!crashing`. All of it is off with FL_FAR_LOOK=1.

## Numbers (deterministic scenarios, bee555e baselines; polled gates 29/29 equal on both pile scenarios and 19/19 on DIR at every step)

Wide line (100 files at ease) hit by the 12-file block; men out of formation at 10/15/20/25/30/35 s, victims alive at 55 s, victims more than 8 m from any attacker at 20/30/40/50 s:

| build | out of formation | alive 55 s | away |
|---|---|---|---|
| bee555e | 74 / 270 / 357 / 390 / 401 / 394 | 282 | 259 / 196 / 152 / 122 |
| mass map + slot face (dc41a22 era) | 51 / 418 / 464 / 449 / 435 / 412 | 323 | 295 / 234 / 195 / 151 |
| field, blurred seeds (rejected) | 85 / 409 / 411 / 396 / 382 / 371 | 341 | 338 / 323 / 308 / 303 |
| field, seed cells + centroid | 70 / 433 / 465 / 451 / 424 / 407 | 317 | 281 / 254 / 248 / 233 |
| field, men's boxes (5639239) | 62 / 407 / 417 / 398 / 368 / 344 | 243 | 273 / 184 / 133 / 79 |

The last row has fewer stragglers than bee555e from 30 s on, and the line loses more men, since the attacker crashes home for the full 8 s cap (a 42-rank block) and the wrap brings more of the line into contact. The wave is still about twice bee555e's speed (FL_JOIN_REACT and FL_GO_MEMORY untouched).

Two on one, victim alive at 15/25/35/45/55 s: bee555e 459/376/304/244/186; 5639239 420/329/257/193/129 (the attackers crash for 148 ticks, stalled by the centroid rule). DIR at 38 s: kills front 426 side 223 rear 664 vs 406/175/653, control alive 144 vs 174, test 0 vs 0, lone 43 vs 92; damage per hit by sector unchanged.

Cost at 200k (FL_AUTOSTART, 100 s runs, last 10 windows, cycle counters; three alternating pairs, then the same 5639239 binary with FL_FAR_LOOK=1, then main's comparison build; the box otherwise idle, all runs the same day so the absolute level is comparable within the table only: it is about a third above the previous session's numbers for the same binaries, cause unknown):

| build | kernel cycles per soldier | kernel CPU per tick | sight block | soldier update wall | grid |
|---|---|---|---|---|---|
| bee555e | 2861 / 2888 / 2882 | 144.6 ms | 139 to 140 | 9.41 / 9.50 / 9.47 ms | 4.0 ms |
| 5639239 (v2 + field + crash) | 2828 / 2854 / 2852 | 143.8 ms | 44 | 9.50 / 9.58 / 9.58 ms | 3.8 ms |
| 5639239 with FL_FAR_LOOK=1 (bee555e's behavior on the same binary) | 2906 | 146.0 ms | 156 | 9.65 ms | 3.9 ms |
| main | 2517 | 126.5 ms | (no far look) | 8.69 ms | 3.6 ms |

Reading it: the sight block fell from 140 to 44 cycles per soldier as designed and the memo lookups are gone, yet the kernel is only 1% under bee555e, because the men now do more (out of formation sooner, more of them moving, the crash pushing whole blocks, about 210 start events per tick). The polled run shows the plumbing itself costs about 30 cycles over bee555e. Against main the branch stands at +13% (bee555e was +14%): that is the price of the footwork behavior, men moving who stood still on main, not of perception any more. The next perf step is a section profile of 5639239 against main (work/scripts/perf/sect.py needs its anchors updated) to see where those ~330 cycles sit; the terrain samples and the integrate paths of moving men are the first suspects.

## Open

- Gota's feel check on 5639239: the wide line, the two-on-one, the 5 v 5 (blobs gone? the charge crash visible?), the 200k.
- The wave's speed (about 2x bee555e), FL_JOIN_REACT / FL_GO_MEMORY.
- The crash's feel: CRASH_STALL 0.3 m/s, CRASH_PACE 4 m/s, the 8 s cap.
- The cost ABAB at 200k, then a polled run for the behavior/perception split, then main.
- Then new baselines and the movement.rs refactor.

## What the numbers prove, and the way forward (written after Gota's verdict on the cost)

Proven: the perception no longer costs anything that grows with sight or density (sight block 140 to 44, memo lookups gone), and the polled run bounds the v2 plumbing at ~30 cycles. Not proven, only inferred: that the remaining +330 cycles over main are the moving men. A section profile of 5639239 against main is the missing measurement; the sections to time are the touch scan, the perception and commit block, the closing and joining block, and the integrate with its terrain samples.

The way forward on both axes, behavior kept as approved:

1. On this branch, cheap and exact (each bit-identical on the polled gates): fuse the lanes look into the touch scan on look ticks (one pass over the box instead of two, about -25 cycles per man going somewhere); a terrain speed-factor map per collision cell computed once per battle (terrain is static apart from craters, which invalidate their cells), replacing the two bilinear samples every moving man pays each tick (about -60 per moving man); the start-event pass spread across the pool instead of serial on the worker (-0.1 ms wall). Together roughly a third of the gap to main.
2. The rest of the gap cannot be closed by the footwork, because main's own cost is the scan and the update every man runs every tick whether he moves or not, and the footwork makes more men move. Below main, with this behavior, means standing men must stop paying: the tile structure of work/notes/perf-fundamental-2026-09-25.md (rigid tiles reuse last tick's scan, static tiles sleep, dirty tiles rebuild). It composes with the footwork unchanged and is the road to 1M; it belongs on a branch off main after this one merges. Expected at 200k with this branch's behavior: kernel from ~2850 to ~1700, below main's 2517; at 1M, cost that follows the front, not the army.

The order I would take: the section profile first (half a day), item 1 on this branch, the feel pass and merge, then item 2.

## Manual fps check at 200k (2026-09-26)

Written by Claude Opus 5.5.

Gota ran three builds back to back around 12:55 to 13:00 (reflog: the branch tip, then `796ff14` checked out at 12:57, then `main` at 12:59) and read the fps counter at the top left (two-second average) at 200k after the lines started engaging. The camera was handled by hand, so the views were similar, not identical. Loadavg was not recorded. No Astra worktree existed at the time.

| build | fps after contact | frame time at the midpoint | against main |
|---|---|---|---|
| main (cc7b08b) | 125 to 135+ | about 7.7 ms | |
| 796ff14 (first footwork v2 behavior) | 110 to 120 | about 8.7 ms | about -12 % fps, +1.0 ms |
| 5639239 (branch tip) | 90 to 100 | about 10.5 ms | about -27 % fps, +2.8 ms |

Gota's read, in his words: "I think you only made it slower."

Reading it:

- Every step on the branch after 796ff14 lost frame rate. 796ff14 to the tip costs about 1.8 ms per frame on top of 796ff14's own 1.0 ms. The perf commits in between (comrade perception out of the scan, the quarter-second look, the touch box) did not win back what the behavior commits cost.
- The cycle counters above undersell the loss. They put 5639239 at +13 % kernel CPU over main (143.8 against 126.5 ms per tick) and level with bee555e. What Gota saw is about twice that gap. So the cost is not all in the soldier kernel's cycles, or the counters miss the serial parts of the tick. The start-event pass on the worker and the crowd field are the first places to look.
- Next measurement: wall-clock frame time and tick-job wall time for the three builds, same locked camera in steady melee at 200k, loadavg logged. That tells which commits cost the frame, where the cycle counters could not.
