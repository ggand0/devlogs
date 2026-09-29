# 0162: river map vegetation with the authored trees

Written by Claude Opus 5.5.

2026-09-29, while Gota works on audio in the main tree. Commit bd26ae0 "Plant the river map's woods with the authored trees and shrubs" on feat/vegetation, not pushed. Review page tmp/shots/0036_river-vegetation/index.html (before and after from the same cameras).

## Ask

Replace the River map's box trees with the new trees and shrubs, placed with the same care as Grassland. Collisions do not matter on this map, so the woods may stay where they are, middle of the map included. Not a one-for-one swap.

## What changed

- The old `plant()` built flat-shaded box pines, broadleaves and bushes (1,077 plants) merged into one mesh per terrain chunk, on a 6.5 m jittered grid where `fbm(p / 100 + 211.3)` ran high: trees above 0.55 past 200 m from the centre, bushes above 0.47 past 150 m, pines on ground above 7 m, clear of the river corridor, heights 1 to 13 m. It and its helpers (`add_plant`, `Kind`, `push_transformed`, `xform`, `Soup::pyramid`, `Soup::is_empty`) are gone; `Soup::cuboid` and `srgb` stay for the bridge.
- `river_planting` keeps the same noise, radii, height band and river corridor, so the woods stand where they stood, and reads the noise depth as structure: a core above 0.62 (90% of cells; 40% mature, 35% upright, 15% leaning oaks, 10% birches), an edge from 0.55 (70% of cells; 35% leaning, 25% birches, 25% upright, 15% mature) with zero to two shrubs under each edge tree, and a scrub fringe from 0.47 (groups of one to three shrubs in 36% of cells, a lone leaning oak or birch in 4%). Ground above 8 m turns 15% of trees into birches, where the pines took over before.
- Banks: a candidate every 7 m along both banks, 45% taken, clear of the bridge by 18 m; groups of one to three shrubs, or past 200 m from the centre an 18% chance of a birch. They sit just past the corridor (2.3 channel half-widths), on the grass above the sandy bank.
- Grid 7.5 m. The authored crowns are about twice as wide as the boxes, and the spacing rule of the Grassland placement thins the cores further.
- The placement code is shared: `try_place` takes a clearance predicate instead of the Grassland `Clearance`, and `Planting::log` takes the map name and its groups. Grassland plants exactly as before: all 296 plant lines of the debug log match the previous build byte for byte.

## Result

352 trees (90 mature, 88 upright, 76 leaning oaks; 98 birches, 28%) and 413 shrubs (328 B, 85 A). Cores 252 trees; edges 72 trees and 58 shrubs; fringe 18 trees and 243 shrubs; banks 10 birches and 112 shrubs. Candidates turned down: 109 for clearance, 115 for slope, 302 for spacing. Tests (both Grassland tests and the three mip tests), strict clippy and the build are clean; no panic or shader error in any capture log.

Tuning on the way, from captures: the first pass (9 m grid, 85/60% fill, high ground 50% birch) gave 277 trees at 50% birch and woods much thinner than the boxes on the ridges; the second (30% high-ground birch) still 38%. The final grid, fills and 15% bring the ridge woods back near the old spread and birches toward the Grassland mix. Bank shrubs went from singles to small groups: singles read as nothing from any distance.

## Open

- From far away (the 950 m overview, the 320 m ridge views) the woods read paler than the dark boxes did: past 280 m no shadow, and the far cards. Known limit (0160); canopy shading in the assets is Astra's item.
- 765 authored plants, each a parent and two part entities, against 296 on Grassland. No battle measurement on this map; it goes with the Grassland 200k measurement if Gota wants River in it.
