# 0212: Men shuttling between their slot and an enemy at the unit's side

Written by Claude Fable 5.1. Date: 2026-10-10. Branch feat/soldier-scale at 46bb464, read only; nothing built or run. Gota's screenshots: refs/foot-work-swing/debug0.png and debug1.png.

## What Gota saw

When an enemy stands at a unit's side and a detached blob of the unit forms there, some men go toward the side enemy a little and then back toward the main block, over and over, for as long as the fight lasts. Gota asked whether this comes from the footwork logic (PR #9) or from this branch, and how to fix it; the fix goes on another branch.

## The cause, by reading the code (a hypothesis, not measured)

A man whose unit's melee clock is not running is under two pulls at once, and neither wins for good:

- His slot pulls him back (`drive`, src/sim/soldier.rs:570): at least 0.8 m/s once he is 0.7 m off it, growing 0.06 of his speed per metre (0.57 m/s per metre for a man-at-arms).
- The enemy he remembers pulls him in (`close_in`, src/sim/soldier.rs:1432): the chase is allowed for a man in formation whenever his unit is not in melee (`!in_melee || committed`). It reaches 5 m from the enemy, 16 m while anyone of the unit is striking (src/sim/soldier.rs:1455). It is a jog of 2.87 m/s beyond 3 m from the enemy and a walk of 1.06 m/s inside (src/sim/soldier.rs:1485). In reach after a swing it is nothing, because he is at his fight distance.

So he jogs in, the chase drops to a walk at 3 m, the slot drags him out, the jog returns, and so on. Figure work/notes/vis/045-slot-and-chase-shuttle.png (script work/scripts/viz/slot_chase_shuttle_045.py) draws the two pulls for a man-at-arms whose slot is 6.5 m from the enemy: the shuttle is about 1.5 m around the 3 m line. The other flips of the chase give longer ones: the 16 m to 5 m change when the unit's striking pauses for 1.5 s (ENGAGE_HOLD_TICKS, src/frontline.rs:30), a swing (the pull is nothing at the fight distance), a comrade stepping into or out of his way, and the crowd at the blob's rim.

The footwork's cure for the run-back is `out_form`: a man out of formation has no slot pull. It only exists while his unit's melee clock runs (`committed = in_melee && out_form`, src/sim/soldier.rs:462; cleared at 453), and the clock starts only when 3% of the unit, at least 4 men, are in their wind-up on the same tick (src/frontline.rs:319). A wind-up is about a fifth of a fighting man's time, so about 15% of the unit must be fighting at once. A thin fight at the unit's side never gets there, so the unit stays out of melee and the shuttle has no end. At the rear the same loop runs, with one difference: a man in formation never swings at a man behind him, so there he lunges and is dragged back without a blow (devlog 0130's guess).

The same symptom was seen on the footwork branch before this branch existed: devlog 0130, item 2 (2026-09-26), "some men went back and forth between their slot and the approach", left unpinned in handoff 041.

## What this branch changed inside the loop

Nothing of the two pulls, the 3 m pace change, the 5 and 16 m chase reach, or the 3% clock. The branch changed the crowd at which the chase gives up (from a real press, CROWD_SLOW 1.2, to the brake's start, 0.43, six neighbours at 1.3 m), the distance he closes to (0.9 of his reach instead of 1.2 m) and added the step back for men out of formation. The wider fight distance puts fewer men in reach at once, so the 3% clock may start less often on this branch than on main; that would make the shuttle show more, not create it. Not measured.

## The one check that settles the branch question

Play main's binary (tmp/runs/soldier-scale/bin/flanks-main) in the same situation and look for the shuttle at a unit with an enemy on its side: one play, no build. To pin the mechanism instead: a temporary switch that gives a man out of melee who chases an enemy no slot pull, one build in target-agent (about 3 min) and one play.

## The fix proposed (for its own branch)

The man's own state decides which pull he follows, not his unit's clock. `out_form` becomes his own: he leaves formation when he sets off for an enemy he sees (the chase's own start, with the melee join's 20% roll per half second so a side enemy pulls men out one by one) or strikes one; out of formation he has no slot pull and chases, fights and steps back as a committed man does now; he comes back to his slot when he sees no enemy any more (nobody in reach, no remembered enemy in sight), or when his unit is ordered to move, holds, routs or re-forms after its melee. The unit's clock, frame, join wave and patience stay as they are for a real melee.

Removals: the chase for a man still in formation (it becomes his departure). Risks for the feel: a side enemy standing 4 m off a flank keeps a handful of men out of the ranks for as long as it stands there, fighting it; a unit left standing after its melee keeps the men who still see an enemy out of the re-formed ranks.
