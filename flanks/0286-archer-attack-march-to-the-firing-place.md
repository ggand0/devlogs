# Archer attack march: the firing place, the hold before a block in the way

Written by Claude Fable 5.1. Branch `feat/archer-rework`, 2026-10-23. Handoff 144 (Gota's bug C of devlog 0283, pictures `refs/archer/attack_move_debug0.png` to `debug3.png`, `attach_move_debug4.png`, `attack_move_debug5.png`).

## The bug

An archer regiment ordered to attack a distant block, with a friendly block across its path, streamed round the friend into a long column. The column's head walked on into the enemy while the regiment's centre still read "out of range"; when the regiment finally stood off, the men walked back, and a few of the head were caught in melee. Without a block in the way the back-off came earlier but was still there.

## The cause

Every man under an order walks to the order's point plus his slot offset (`drive`, sim/soldier.rs). For an archer attack order that point was the enemy block's centre (`Groups::goal`), so every man's destination lay inside the enemy, and only a separate test on the regiment's rectangle about its centroid stopped the march, at the close-in gap. The rectangle about the centroid is not where a column's men are: the centroid lags far behind the head. When the test fired, the frame snapped to the centroid, so the head walked back to slots behind it. The hold rule that could have stopped the head before an enemy block looked 50 m ahead of the same centroid, and ran only with skirmish on.

## The fix

- **The march's destination is the firing place** (`firing_place`, sim/job.rs): the frame centre on the line to the target from which the front rank stands 0.8 of the bows' reach short of the target's near edge, kept inside the map, recomputed every tick as the target moves. `Groups::goal` returns it for a bows-out archer regiment, so the men's slots, the route planner and the heading all use it. No man's slot lies in the enemy, and a column's head stops at its slot like everyone else.
- **The stand-off is the leading men's arrival** (`head_arrived`): the foremost living man along the facing within the Move order's arrival distance (18 m) of the front rank's line at the place. The centroid would wait for every straggler caught in the friendly block. Entering the stand-off pins the frame to the place and deals the slots in the men's present order (the reform), so nobody walks back; the rest close up as a holding regiment dresses.
- **The regiment's own position for "in range" is its frame** (`frame_in_bow_range`, `formation::block_rect_at`): while standing, the rectangle about the anchor; on the march, about the place. The rectangle about the centroid read 124 m while the body stood 96 m from the target, because a few men stuck in the friendly block dragged the centroid 28 m back.
- **The range is a property of the unit type** (`UnitTypeParams::bow_range`, 0 for the melee kinds, `bow_range` and `bow_reach`): the firing place, the range test, the per-man reach (`pick_quarry`), the skirmish scope, caps and retreat distance all read the shooter's kind. The global `missile::RANGE` and `REACH` are gone; `missile::REACH_FACTOR` (1.2) stays.
- **The hold before an enemy block in the way** (`skirmish_reactions`): the strip ahead starts at the leading men (`GroupData::lead`, the foremost man's distance ahead of the centroid along the facing, summed in frontline.rs), so a column is seen by its head; it runs for an attack order's close-in march with skirmish off too; a holding regiment looks 15 m further (the hysteresis); the block found is kept (`skirmish_hold: Option<u32>`) and is what the regiment shoots, whatever its order and fire at will, unless a block is coming for it. A beginning hold pins the frame with its front rank on the leading men and deals the slots (`pin_frame_at_head`); before, the hold left the anchor where the march began and the whole regiment walked back to it.
- **A standing-off regiment never holds** (it has no path), and a holding regiment keeps heading for its goal (the hold asks whether walking on would run into the block); without that the hold flickered every tick, since a standing regiment's velocity is zero.
- **A hold that ends keeps the order**: `end_skirmish` restored the parked order for any regiment, so a regiment that had only held (nothing parked) lost its attack order when a melee began or skirmish was toggled off.
- The run's log line names its trigger and block: `runs (skirmish): block 6 coming, gap 53.0 m`.

## The scene

`FL_ARCHERY_BLOCKED=1` in `spawn_archery_test`: two archer regiments (500 each) ordered at an orange block 310 m off, a friendly block of 500 across their path, a block of 100 orange in the way short of the firing place. `FL_ARCHERY_NOSKIRMISH=1` for Gota's case. The log adds per regiment the lead, the nearest man's distance to the target's edge and to the block in the way, the men in melee, the hold block, and the centroid's and anchor's distance to the firing place.

| t | what happened (skirmish off; skirmish on is the same to the second) |
|---|---|
| 0 to 8 s | the men stream round the friendly block; the lead grows to 31 m; nearest man 60 m from the block in the way |
| 8 s | both regiments hold ("stands and shoots at block 4 in its path"); the frame pins at the head; 187 and 167 drawing |
| 8 to 24 s | the block in the way shot down, 100 to 0 (the archers' nearest man 60 to 76 m from it, the body closing up: lead 31 to 13 m); 0 in melee |
| 28 s | "walks on"; the march resumes, place 127 m off |
| 40 s | both stand off (H) with the lead 44 m and the place 40 m off: the head is there, the anchor on the place; 0 in melee |
| 44 to 104 s | the target shot down, 500 to 0, both regiments volleying; nearest man 94 to 104 m from the target throughout; 0 in melee |
| 108 s | "attack target 5 destroyed, holding" |

Hold messages: 2 holds, 2 walks on, no flicker (the first run had 3859, from the velocity heading). Skirmish on: identical numbers, no run (the block in the way is static).

The other scenes against devlogs 0264 and 0283 (opt-dev, 500 a regiment, 60 s):

| Scene | Result |
|---|---|
| base | both volley, 0 blue struck, the middle block 500 to 0 by t=60 |
| order | stand-off at t=4 for both, no march; east 500 to 42; reg 1 out of range at t=36 as the block shrank, marched, stood again by t=44 (devlog 0264: t=44 and t=56) |
| melee | the men search the spear block's open side; 250 blue struck of 3648 by t=60; see the open item below |
| attack, skirmish on | reg 1 ran at t=1 (116 m, "block 6 coming, gap 55 m"), cornered at t=30, stood and shot; blue struck 33 (devlog 0264: the same shape) |
| attack, skirmish off | the archers volleyed the middle block until the attacker closed, then the melee; blue struck 7 (devlog 0283: 7) |
| swords | charge and melee as infantry, no F, no H |

## Checks

- opt-dev build, strict clippy (`--all-targets -D warnings`) and the 50 tests pass.
- `FL_TEST_DIR` fingerprint, 45 s, against main's binary: identical, 21 fingerprints, last `1260 3222 99dc6fe27e005c91`. The melee kinds are untouched.

## Gota's play of the fix (2026-10-23) and the second round

Gota played 8908bae and found two more (pictures `attack_move_debug6.png` to `debug10.png`): an archer regiment ordered at an enemy block far off to the right, part of the regiment caught in a friendly block, the rest streaming past its end. One: the regiment marched on, began to fire, then suddenly walked back about 40 m and formed up next to the friendly block. Two: standing and firing, it shuttled back and forth by small steps, the card reading "Marching" and "Firing missiles" by turns, with skirmish on a little too. Gota had told Opus of the first before the fix; it was not in handoff 144.

New scene `FL_ARCHERY_SIDE=1` (the friendly block across the left part of regiment 1's path only, the target far right and ahead, a closer enemy block off the path; `=2` has the target walk away slowly). The log gained a line for every stand-off change with the gap, the reach, the place's distance and the lead.

**The cause, from the scene's log** (reg 1 at t=80): `marches at (gap 113.5, reach 104.3)`, 30 ms later `stands off (reach 112.5)`, 70 ms later `marches at (reach 74.6)`, then `stands off (reach 112.8)`. The reach is the range plus three seconds of the target's speed toward the regiment, and that speed was the track of the target's centroid, which jumps whenever men of a shrinking block fall: a block of 25 men losing 10 to a volley moves its centroid metres in one tick, which the track read as tens of m/s. Every flip re-entered the stand-off, re-pinned the frame and re-dealt the slots: Gota's shuttle. The firing place carried the same term, so a target moving toward the regiment, or a jumping centroid, moved the place backward and the whole regiment walked back after it: Gota's sudden walk back. The old code's range test had the same term, which is why the walk back was there before the fix too.

**The fixes:**

- `RegTracks::vel`, the volley lead and the range tests' closing speed, is the mean velocity of the regiment's living men (`GroupData::vel`, summed in frontline.rs), not the centroid's motion. The aim lead on a dying block was wrong by the same jumps.
- The firing place is laid on the standing reach (`bow_reach_standing`: range plus height), never on the target's speed. The lead term stays in the range test only, as the reference keeps it.
- A regiment without an order has no path of its own, so it never holds before a block (the hold fired on a dressing drift after the target died, with a pin and a re-deal).

**The side scene after the fixes** (skirmish off, on, and the walking target): one stand-off per regiment, no flip, no walk back; the target shot down from 500 to 0 by t=116; the walking target followed and stood off once at its new place. The blocked, order, attack and no-skirmish attack scenes are as before.

## Open

- **Men caught in a friendly block** (Gota's picture `attack_move_debug5.png`): an archer pushing through a holding friendly block is stopped inside it by the crowd yield and stays. The fix no longer waits for them (the arrival and the range test read the leading men and the frame), but they are still stuck. That is the "allies make space" item of the roadmap, its own branch.
- **Sliding and moonwalking** while dressing: the shuttle itself was the stand-off flip above. What remains is the look of a slow dressing step: a bow-out archer has no step clips. The record's fourth position float carries the bow, and the knight's shuffles play on him only with the knife out (unit_build.wgsl, devlog 0283), and the step clips play only in the ready stance anyway (`band`, devlog 0207). A bow-out man stepping back to his slot at the shuffle pace plays his walk clip facing forward. Bow-out shuffles and the packing to carry them beside the bow are Astra's and the renderer's.
- **The run flickers against a block grinding in melee** (the melee scene, reg 1 at t=16: "runs: block 6 coming, gap 53 m" and "ends its run" every tick for a second, then again at t=59). The approach trigger (`approach_hits`, stage 2 of devlog 0264, untouched here) takes a block moving at the walking threshold (0.3 m/s) in a melee as coming, with a path a few metres long that still overlaps the two wide rectangles. The pre-fix tip (abb2ff6, built from a `git archive` export in `tmp/runs/archer-rework/tip-src`) does the same in the same second: 27 run and end pairs against 17 on the fix (`scene-melee-tip.log`, `scene-melee-fix.log`). Not this fix's; a defect of the stage 2 trigger for its own commit.
- **A cornered regiment toggles** "runs" and "is cornered" every 15 ticks while the melee shoves it about the map corner (the attack scene, t=35 to 40): the retreat point falls inside and outside the arrival distance by turns. Pre-existing from stage 2.
