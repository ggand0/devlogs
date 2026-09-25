Written by Claude Opus 5.5

# 0125: Far sight from a field

Branch `feat/melee-footwork`, after bee555e (devlog 0124). Gota: overhaul the perception so it does not explode the compute while the soldiers stay realistic.

## The problem

After devlog 0124 the largest remaining cost of the footwork is the sight scan. Every 8 ticks, a man of an engaged regiment with no enemy in reach checks every unit in a 30 x 30 m square around him (about 500 in a dense battle) and keeps the nearest enemy as the one he goes for. That one scan does two jobs:

1. noticing enemies far away, up to 15 m (whether an enemy is in sight at all, and which way);
2. finding the enemy right in front of him, the man he actually walks up to and fights.

Its cost grows with the square of the sight range. A 1 s cadence for the whole scan was tried in 0124 and delayed the melee start by about 2 s, because job 2 got slow (Gota has not seen that run; it stays unverified by him).

## The approach: three ranges

Picture: tmp/notes/vis/perception-tiers.png (today on the left, proposed in the middle).

- Touch, every tick (unchanged): the separation scan, bodies and an enemy in weapon reach. Its neighbors, every 8 ticks: lanes (way blocked, open sides, comrade ahead, mark blocked) and comrades running close by.
- Near, job 2: the existing 4 m approach scan (WIDE_ACQUIRE_R, about 50 units), run only when the far look says an enemy may be within 4 m. It picks the actual nearest enemy.
- Far, job 1: a coarse field of 4.5 m cells, rebuilt once per tick, in which every cell stores the nearest living soldier of each team and the nearest comrade running out of formation of each team. A man reads the cells around him: the nearest enemy and how far, whatever the sight range.

Picture of the field: tmp/notes/vis/sight-field-proposal.png. The center man of a wide line reads an enemy 5 m ahead (in sight; his near look then picks the man); a flank man reads 25 m (beyond sight; he joins when he sees comrades run, or when his patience runs out, as before).

It stays local perception: a man reads only the spot he stands on, the precomputed answer to "where is the nearest enemy from here". Nothing regiment-wide.

## Cell size

Picture: tmp/notes/vis/perception-tiers.png, right panel.

- 4.5 m is 3 x 3 of the 1.5 m neighbor-grid cells, so the field is built from the grid's sorted cells without another sort.
- A cell stores the soldier nearest its center. For a man off-center that soldier can be up to one cell diagonal (6.4 m) further than his true nearest. Fine for job 1; job 2 is the near look's.
- Cost grows with the number of cells: about 19k at 4.5 m for the 200k battlefield (43k at 3 m), against 200k men.

It is a trade-off, not a derived number: build time and pick error at 4.5 m and 3 m are measured below.

## Concerns and fixes

Picture: tmp/notes/vis/sight-field-boundaries.png.

1. Two men side by side across a cell line would head for different enemies if each read only his own cell (left panel: 5.0 and 5.2 m while the true nearest is at 3.2 m). Fix: read the 3 x 3 cells around him and take the stored soldier nearest to himself (middle panel); both find the same man. Measured on 6,000 random placements behind a ragged enemy line (tmp/scripts/viz/boundary.py; the pictures come from the other scripts in tmp/scripts/viz/):

   | cell | own cell only | 3 x 3 read |
   |---|---|---|
   | 4.5 m | 35% pick another enemy, worst +3.4 m | 12%, worst +2.1 m |
   | 3 m | 23%, worst +2.2 m | 5%, worst +1.3 m |

   A smaller cell helps, but the 3 x 3 read is the bigger fix. The leftover error never matters close in: within 4 m plus a cell diagonal the near look picks the exact nearest.
2. The neighbor grid's corner sits on its outermost unit, so every cell line on the field drifts each tick with that one man (right panel), and the field would re-pick targets for nothing. Fix: snap the grid to world multiples of 4.5 m. This also stops the drift of today's touch-scan boxes: a small behavior change of its own.
3. The 15 m sight edge becomes approximate by up to about 2 m (which enemy he measures can be off). Only men right at the limit notice.
4. Comrades running: the field stores the nearest runner of each team, and "running" becomes the runner's own state (out of formation and faster than 1 m/s), what an onlooker actually sees, instead of his speed projected on the onlooker's own way to the fight. Per team by default: a man follows any comrade of his side he sees run in. FL_JOIN_OWN_REG=1 counts only his own regiment's runners (the stored runner, and the close-by check on the touch neighbors, must be of his regiment). Gota checks the behavior later.
5. The build cost was an estimate (0.2 to 0.5 ms per tick on the sim thread): measured below.

## Build

Built as described (uncommitted working tree on top of bee555e; patch in tmp/patches/sight-field.patch):

- spatial.rs: the grid origin snaps to world multiples of FAR_CELL; META_RUNNER (bit 7, out of formation and faster than 1 m/s; the regiment moves up to bit 8); `build_far` at the end of every rebuild: parallel seeding over bands of 8 coarse rows (each coarse cell takes the unit nearest its center per layer from its own 3 x 3 grid cells), then per layer a forward and a backward raster sweep over the 4 visited neighbors (the 4 layers in parallel); `far_nearest(layer, p)` reads the 3 x 3 cells around p.
- movement.rs: a man of an engaged regiment on his far look (the acquisition's 1-in-8 rhythm, same gates) reads the enemy layer; within sight the named enemy becomes his target, and when it may be within the 4 m approach scan's reach (4 m plus a far cell diagonal) that scan picks the actual nearest. The approach (not engaged) keeps the old 4 m scan. The far comrade-running check reads the runner layer (within 6 m, not himself); the close-by check on the touch neighbors uses the same META_RUNNER. FL_JOIN_OWN_REG=1 restricts both to his own regiment; default per team.

Behavior against bee555e (same deterministic scenarios): wide-line roll-up out of formation at 10 / 15 / 20 / 25 s 68 / 273 / 346 / 383 vs 74 / 270 / 357 / 390; two-on-one victim alive at 55 s 194 vs 186; DIR at 38 s front / side / rear 420 / 204 / 629 vs 406 / 175 / 653. Tracks it.

Pick quality in the 200k battle (FL_AUTOSTART, 100 s, every far look compared against the full 15 m scan): the sight edge agreed on every look (no look where one side saw an enemy and the other did not). The enemy picked was another than the true nearest in 45% of looks at 4.5 m (mean 0.72 m further, worst 4.7 m) and 37% at 3 m (0.53 m, worst 3.0 m). In a dense enemy crowd many men stand at nearly the same distance.

Cost at 200k (kernel cycles per soldier, last 12 windows of 5 s; grid is the rebuild wall time per tick, field included):

| build | kernel cycles/soldier | kernel CPU/tick | step wall | grid wall |
|---|---|---|---|---|
| bee555e | 2162 | 109 ms | 8.09 ms | 4.25 ms |
| field 4.5 m | 2119 | 107 ms | 8.13 ms | 6.04 ms |
| field 3 m | 2094 | 106 ms | 8.07 ms | 7.50 ms |

With a counter on the sight block alone (both builds, same instrumentation): 133 to 179 cycles per soldier before, 64 to 71 with the field, so about 85 saved (4 ms of CPU per tick). The field build at 4.5 m (218 x 88 cells): seeding 0.42 ms wall, sweeps about 1.0 ms wall and 2.8 ms of CPU over the 4 layers. It costs about what it saves, and it adds about 1.5 to 2 ms to the tick's wall time, because the build runs before the integrate.

## Conclusion

At 200k with 15 m sight the field is a wash. The old sight scan was only about 115 cycles per soldier of the 2162 (about 5% of the kernel), so even a free field could save at most about 0.4 ms of step time. The estimate of 0.2 to 0.5 ms for the build was wrong by 3 to 4 times. What the field does buy is that its cost follows the battlefield's area, not the sight range: at 40 m sight (M2TW's AI engage distance) the scan would cost about 7 times as much and the field the same.

The remaining gap to main (2162 vs 1780 cycles per soldier) is spread thin: sight about 115, perception at its quarter-second rhythm about 35, the remembered-enemy lookup and the join logic about 130, and about 100 in sections whose code is identical to main's (the battle itself: more men moving).
