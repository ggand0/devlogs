# Big Grassland benchmark at the wrong window size, and the packed army switch
Written by Claude Opus 5.5.

2026-10-03. Gota asked whether Astra's Big Grassland (`feat/big-grassland` 809ef2a, devlog 0180) is fine for performance, and for a benchmark of the map's drawing cost. Then for a test mode that packs the 200k armies around the centre as on the old map, since the default layout spreads them thin and most draw at L2 and L3.

## Void as benchmarks: every run was at 1600 by 900

All three batches below ran without `FL_WINDOW`, so the game opened at the saved Windowed default: 1280x720 logical at 125%, 1600 by 900 physical. The benchmark window since devlogs 0170 to 0174 is the maximized 2K one, `FL_WINDOW=2560x1360`. The level bands follow the window's height (devlogs 0168, 0170), so at 900 rows most of the army draws at L2 and L3, the GPU passes are a fraction of their 2K cost, and the frame reads about 200 fps and looks CPU-bound. Gota: "it's cheating, no longer useful". I did not read the perf devlogs before designing the runs and went by memory, which did not name the window. The rule is now in memory (measuring-perf) and `map-bench.sh` defaults to `FL_WINDOW=2560x1360`.

What still holds, because it does not depend on the window:

- The density field (`InfluenceField`, `src/frontline.rs`) covers the whole map in 8 m cells: 12.5k cells on Grassland, 66k on Big Grassland. Its time per tick goes from 0.4 to 0.7 ms to 1.7 to 2.5 ms, with 4,000 men or 200,000, since it scales with the area. The fixed tick on the main thread grows about 0.3 ms (p50 1.5 to 1.8), so most of it runs alongside other work.
- The grid rebuild did not grow (about 2.0 ms on both maps at 200k).
- The branch's Grassland matches main within run noise at 200k (195.8 fps against main's 198.5 and 196.1).

Not known until the 2K runs: the ground pass and unit pass costs on Big Grassland, and whether the frame is GPU or CPU bound there.

## Setup

Main 25ec337 (`target/opt-dev/flanks`, md5 aba27e09) against Astra's branch build (`../flanks-gfx/target/opt-dev/flanks`, md5 f7b3b938) and the packed switch build below (md5 49596d22). Both tree builds match their source: no file in either tree is newer than the build, and the graphics tree is clean. Copies as `flanks-main`, `flanks-bigg` and `flanks-field` in `tmp/runs/map-bench/bin`, each finding its assets in its own tree. Script `work/scripts/perf/map-bench.sh` (`terrain`, `battle`, `packed`), summary `work/scripts/perf/map_bench_summary.py`. GPU clock 1875 to 1950 MHz in every run. Loadavg 1.7 to 6 in the first two batches and 6 to 12 in the packed one (a `cargo test` build had just ended; nothing heavy showed in `top` after it). Gota's three viewskater windows were open on the GPU throughout. One run per cell.

## The packed army switch

`FL_ARMY_FIELD=WxD` (metres) makes the army layout fill a field of that size centred on the map, clipped to the terrain, instead of the whole terrain. `FL_ARMY_FIELD=1024x768` gives Grassland's exact packing on Big Grassland: 13 units per rank, 8 ranks, a 74.15 m pitch, the line ends at about x = ±482. The width alone is not enough: on Grassland the depth limit (8 ranks) is what forces 13 per rank, and Big Grassland's depth would allow 12 per rank in 9 ranks. The usual views (x ∓470 ground, x ∓400 mid-air) then apply unchanged.

Commit 4531baf "Add a switch to pack the armies into a smaller field" on the new local branch `feat/big-grassland-bench`, on top of 809ef2a. Made with a temporary index (`git hash-object`, `read-tree`, `update-index`, `write-tree`, `commit-tree`, `git branch`), since `feat/big-grassland` is checked out in the graphics tree; neither working tree changed. One file, `src/regiments.rs`: `army_field()` returns the field's corners, the terrain's own when the switch is unset, and `do_spawn_battle` takes `usable_w` and `usable_d` from them. Commit 17043d4 "Fix the army layout docs" on top moves the clamp reason in the layout comment to the terrain's edge, where it belongs.

Checks:

- Strict clippy on all targets clean; 35 tests pass.
- Switch unset, fingerprints against Astra's build, static 200k (`FL_AUTOSTART=1 FL_DEPLOY=0 FL_AI=0 FL_ENEMY_STATIC=1 FL_HEAVY_FRAC=0.4 FL_HASH=30`, 30 s): Big Grassland 27/27 equal, Grassland 27/27 equal, Astra's build against itself 27/27. The first try without `FL_HEAVY_FRAC` mismatched even build against itself: a normal battle draws a random enemy army style per launch (87, 73, 95 heavy units in three runs). The test front uses the fixed split, so the benchmarks were not affected.
- Switch set: screenshots at 1700 m, pitch 1.2, in `tmp/shots/army-field/`: `packed_12s.png` shows each army 8 rows deep and about 1 km wide in the middle of the map; `default_12s.png` shows 4 rows across the full 2 km. In the packed battle 174.7k soldiers are drawn in the ground views, as on Grassland (174.5k).

## The packed battle, at 1600 by 900

`FL_TEST_FRONT=1 FL_UNITS=100000`, AI on, 120 s, mean of the last 60 s. Ground views pitch 0.35, distance 15; mid-air pitch 0.28, distance 140; west yaw -1.5708, east +1.5708. Run order: west main, west packed, east packed, east main, mid-west main, mid-west packed, mid-east packed, mid-east main.

| View | main, Grassland | Big Grassland packed | Field ms, main / packed |
|---|---|---|---|
| Ground, x -470 | 204.4 fps, 4.90 ms | 193.0, 5.18 | 0.65 / 2.15 |
| Ground, x +470 | 194.2, 5.15 | 191.1, 5.24 | 0.67 / 2.25 |
| Mid-air, x -400 | 208.8, 4.80 | 195.1, 5.13 | 0.70 / 2.09 |
| Mid-air, x +400 | 204.1, 4.90 | 199.1, 5.02 | 0.67 / 2.51 |

Drawn soldiers match (174.5k against 174.7k on the ground, 179.4k against 179.7k mid-air) and so does the unit pass (3.23 to 3.43 ms ground, 2.33 to 2.48 mid-air), so the packing works. At this window the frame is about 0.1 to 0.35 ms longer on Big Grassland, in line with the field.

## The first two batches, at 1600 by 900

Stretched layout, the 200k battle (same settings, the flank view 42 m inside the west edge: x -470 on Grassland, -982 on Big Grassland):

| Run | fps | Frame ms | Ground pass | Unit pass | Sun shadows | Grid | Field | Drawn | Fixed tick p50 |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| main, Grassland | 198.5 | 5.04 | 0.13 | 3.23 | 0.31 | 2.11 | 0.65 | 174,545 | 1.5 to 1.6 |
| branch, Grassland | 195.8 | 5.11 | 0.13 | 3.24 | 0.30 | 2.06 | 0.67 | 174,548 | 1.5 to 1.6 |
| branch, Big Grassland | 188.2 | 5.32 | 0.46 | 2.05 | 0.20 | 1.99 | 2.17 | 157,894 | 1.7 to 1.9 |
| main, Grassland again | 196.1 | 5.10 | 0.13 | 3.24 | 0.29 | 2.14 | 0.71 | 174,545 | 1.5 to 1.6 |

The stretched line is about twice as long and half as deep, so fewer men are in view and more of them far away: the unit pass is 1.2 ms cheaper. That is the thin spread Gota saw in play.

The ground alone: 2,000 men a side, standing, 40 s, mean after 15 s. Grassland draws its trees and shrubs; Big Grassland has none yet.

| View | main, Grassland | branch, Grassland | branch, Big Grassland |
|---|---|---|---|
| Flank, x -470, distance 15 | 295 fps, 3.39 ms, ground 0.45 | 298, 3.35, 0.45 | 280, 3.58, 0.55 |
| Mid-air, x -400, distance 140 | 297, 3.37, 0.63 | 291, 3.44, 0.63 | 279, 3.59, 0.82 |
| Overview, distance 900, pitch 0.9 | 314, 3.18, 0.38 | 301, 3.32, 0.39 | 295, 3.39, 0.64 |
| Whole map, distance 2800, pitch 0.9 | | | 252, 3.98, 0.92 |

The ground pass grows with the ground in view (the horizon reaches about 2.9 km with the 6000 m far plane; the whole map is 1,024 chunks and 2.1 million triangles). It is mostly pixel work, so it is larger at 2K by an unmeasured amount.

## Next

- The 2K runs: `map-bench.sh packed` now runs at `FL_WINDOW=2560x1360`, the four views in 8 runs of 120 s, about 17 minutes.
- The density field over the whole map: cover the area the armies hold, not the map. A proposal for Gota first; the front line and the AI read it.
- PR draft: `work/drafts/040-pr-big-grassland.md`, its performance line waits for the 2K runs.

## Outcome: merged as PR #24 without the 2K batch

Later the same day (Claude Opus 5.5, the next thread).

- The fast-forward ran: `feat/big-grassland` = 17043d4 (809ef2a, 4531baf, 17043d4). The extra branch `feat/big-grassland-bench` was deleted after the merge.
- The 2K packed batch (`map-bench.sh packed`, about 17 minutes) was started behind a wait for an idle GPU (one viewskater window drew at about 31% GPU) and stopped at Gota's word before any game ran: no time for another 20-minute bench. The void 1600x900 logs moved to `tmp/runs/map-bench/void-1600x900/`.
- Gota ran the packed ground view at 2K instead, from the graphics tree (`FL_WINDOW=2560x1360 FL_MAP=big_grassland FL_ARMY_FIELD=1024x768 FL_TEST_FRONT=1 FL_UNITS=100000`, flank camera at x -470, `cargo run --profile opt-dev`): the fps matches Gota's earlier runs. No numbers were logged.
- The PR's Performance section gives only the density field cost (1.7 to 2.5 ms per tick on Big Grassland against 0.4 to 0.7 on Grassland, fixed tick p50 +0.3 ms at 200k) and "Tested on Linux with an RTX 3090." Gota's rule from this PR: no "I" in PR drafts (docs/internal/002 updated).
- The band grid rebuild: Gota's call, not needed for the big map. The collision grid covers only the box around the soldiers (`src/spatial.rs`), and it cost about 2.0 ms on both maps. I had first called it needed from plan 018's wording, with no measurement behind it.
- Final checks on 17043d4: the full diff against main read, strict clippy with `--all-targets -- -D warnings` clean (a fresh check of an exact copy), 35 tests pass, commit messages, added lines and the PR body free of footers, slop and banned words.
- PR #24 merged: main = 66044d1.
