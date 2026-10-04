# 0190 Pathfinding: routes round obstacles, and the obstacle map

Written by Claude Opus 5.5.

2026-10-04. Branch `feat/pathfinding`, cut from main 66044d1 in `../flanks-agent` (the third tree, linked like flanks-gfx; rule 12 in doc 003). Unattended session from handoff 090. Settled with Gota before the start: the shared-gap test uses two units of the same side passing each other; routes are on for every map with `FL_PATH=0` to turn them off; the obstacle map is launch-only; the perf run happens once pathfinding works, whatever else the machine is doing.

## What it does

A unit ordered somewhere it cannot walk straight to now gets a route round what is in the way: round an 80 m block, through the one gap in a wall that leads there, out of a U pocket the long way, through a zig-zag of three turns, between posts, and over the river by the bridge when that is faster than wading. A soldier whose straight way to his slot is blocked follows his unit's route from where he stands, so the whole unit arrives, not only its centre. A goal nobody can reach (inside the closed box) becomes a move to the nearest ground with room for the unit; the unit stops there and the log states the goal cannot be reached. An order whose straight way is clear behaves exactly as before, and maps with no blocked ground and no water (Grassland, Big Grassland, Classic, Sandbox) build nothing and plan nothing: the scripted battles on Grassland, Big Grassland and River give main's fingerprints, and the 200k battle costs main's per tick.

Commits, in order (`git log 66044d1..`):

- 9eb0680 Add an obstacle map for pathfinding tests
- 473001f Add a walkability grid and a route planner for regiments
- a7d2f1b Give regiments routes around blocked ground
- 378bb61 Add scripted pathfinding scenes
- 39e7db9 Rewrite slots between the install and the kick on arrival and in the scenes
- 9736e92 Build a missing cost table in a tick of its own
- 240945b Fix the route module docs
- 488f1d6 Count the waiting regiments from the tick after the orders
- 576b8fd Fix the blocked ground docs

New files: `src/obstacle_map.rs`, `src/nav.rs`, `src/sim/route.rs`, `src/path_test.rs`. Edits elsewhere are small and in one place each, for the merge with `feat/soldier-scale`: `drive` in `sim/soldier.rs` (plus one field in `Field`), `resolve_orders` and `prepare_tick` in `sim/job.rs` (plus four `TickJob` fields), the job plumbing in `sim/mod.rs` (`run_tick_job`'s plan step, the hand-back at the install, the `Routes` param of `step_sim` and `kick_tick`), the map in `terrain.rs`, one match arm in `vegetation.rs`, the scene hook in `regiments.rs` (`spawn_regiment` made `pub(crate)`) and `game_state.rs` (`FL_TEST_PATH` in `scripts_active`), and `main.rs` (modules, the scene plugin). No soldier column and no `sight` bit were added (`feat/soldier-scale` takes bit 7 and widens `sight`).

## The design

### The obstacle map (`src/obstacle_map.rs`)

`FL_MAP=obstacles`, launch-only, 1024 by 768 m like Grassland, flat ground at 0 m with every obstacle a 6 m raised block. The terrain's own rule (a 2 m step between neighbouring vertices) blocks every side, so the soldiers' wall slide applies with no new collision code. `MapKind::has_blocked_ground()` turns the mask on for this map and the river map; `blocked_at` skips the bridge exemption on maps without a river (the deck rectangle would otherwise open a hole in any obstacle placed there). A block whose top spans vertices a to b blocks a - 2 to b + 2, so a gap between tops ending at a and starting at b is walkable over b - a - 6 m.

| Case | Layout | Start → goal |
|---|---|---|
| cube | 80 m block, 9 m east of the unit's line so the west way is shorter | (-341, -150) → (-341, 70) |
| wide | one long wall at z -294..-288 across the whole field; its 42 m gap at x -300 | (-220, -350) → (-220, -230) |
| narrow | the same wall's 6 m gap (four men abreast) at x 0 | (0, -350) → (0, -230) |
| shared | the same wall's 14 m gap at x 300; two units trade places through it | (300, -230) ↔ (300, -350) |
| pocket | a U 100 m wide inside, 8 m walls, open toward the unit, goal behind its closed end | (0, -150) → (0, 75) |
| zigzag | an enclosure entered at its south wall, three inner walls turning the way back and forth (corridors about 46 m), goal in the last corridor | (-240, 92) → (-240, 336) |
| box | a closed 60 m room, goal at its centre | (40, 130) → (40, 240) |
| pillars | 4 m posts 16 m apart (6 m walkable between), every other row shifted half a pitch, six rows across the whole zone | (340, -150) → (340, 75) |

Tests: a flood from open ground with diagonal steps reaches no block's top and not the box's room, and every start and goal (bar the box's) is reachable; the gaps measure 21, 3 and 7 open vertices across.

### The grid (`src/nav.rs`)

One cell per terrain vertex, 2 m. The soldiers collide with the nearest vertex's blocked bit, so the grid finds every gap a man fits through; 6 m is the narrowest the map asks for, which an 8 m grid erases and a 4 m grid finds only when its cells line up. Per cell: walkable (not blocked, not in the 4 m edge margin the sim clamps to), cost (the time per metre a man takes there: `(1 + 3 slope²) / wade`, the same factors `drive` applies), wet, clearance (exact Euclidean distance to the nearest unwalkable cell, capped at 64 m, Felzenszwalb's two-pass transform), and a region label (cells joined by the planner's steps). The largest region is the field; every other region is cut off (block tops, the box's room). A crater refreshes the cells round it, the clearance within the cap round them and the regions. Other blocked ground (trunks) goes in `mark_blocked`. Maps with no blocked ground and no water build no grid at all.

### The planner

- The straight check first, on the main thread when the order arrives: if every point of the line from the unit's centre to the goal has clearance of at least the unit's half width across that way, and the line crosses no water, there is no route and the order is exactly as before.
- Otherwise A* on the grid, 8 directions, a diagonal only when both cells beside it are open. Cost per metre of a cell for a unit of half width w:
  - the ground's cost,
  - times the squeeze: the unit's width over the open width on the line through the cell straight across the step, when that is less. A block that pours through less width than it has takes that many times as long. A wall gap counts its one lane; a field of posts counts every lane between them (about 2.7 at a row of posts against 4.8 in the 6 m gap). The open width is the narrowest of the line straight across and the two at 45 degrees, so a step cannot look roomy by crossing a thin wall at a slant;
  - times 1 + 2 x the room given up: where the line across has a spot with more clearance (up to w) than this cell, the share of w given up. This keeps a route in the middle of a passage wide enough for it.
  The two factors are precomputed per unit width (a cost table, about 18 ms on one thread, built on the pool before the plans that need it and shared by every later plan of that width).
- A goal in another region (the box) or on unwalkable ground: the nearest cell of the start's region with clearance of at least w, else the nearest at all. The order becomes a Move there and the log states that the goal cannot be reached.
- The grid path is then pulled straight: from each kept cell, the farthest later cell reachable by a segment that is walkable, keeps the clearance the grid path had (up to w) and costs no more than 2% over that stretch.

### Following the route (`src/sim/route.rs`)

- The unit's frame (the point `resolve_orders` hands its men, and their slots are around it) is the current waypoint; the last is the live goal (an attack target's centre). The frame moves on once the way to the next waypoint is clear and dry for the unit's whole width, or once the centre is within w of the current one. Men keep full pace short of the last leg.
- Each man: his straight way to his slot open (a clearance march over the grid, a few lookups in open ground) → his slot, as before. Blocked → he follows the route from his own place on it (the nearest point of its legs up to the frame): the farthest waypoint ahead he can walk to straight; else the farthest point along his leg; else the nearest point back along it; else he steps round what is in front of him, turning first to the side his place on the route lies. He knows his slot, his unit's route and the ground in front of him.
- On arrival the unit keeps its route while it holds at the route's end (so men still behind the obstacle find their way after the order clears), and its slots are re-sorted by where the men stand (`assign_slots`): coming round obstacles mixes up the ranks, and without this men stood 5 to 9 m from their slots, blocked by comrades on the other side of the block.
- An attack order checks every 15 ticks whether its target has gone behind something (no route yet) or moved more than w off the route's end, and plans again.

### Where it runs

The orders, the straight check, waypoints and adoption are per-unit work in `update_routes`, after the orders are cleared and before the kick. Plans ride in the tick job: at most 16 per tick, planned on the pool after the soldiers' kernel (`util::sim_scope`), handed back at the next install. The schedule is counted in ticks, never in wall time, so runs repeat bit for bit. A unit waiting for its plan (one tick normally, 13 ticks when 200 order at once) walks straight. `drive` reads each unit's route and the grid from the job (`Field::paths`).
## What was tried and dropped

1. A clearance penalty on the cost (`1 + (w - c) / w` below the half width w). The wide-gap route still crossed at the gap's edge (x -280, 1 m from blocked ground): hugging the corner saved 31 m of walking and the penalty cost about 9 m over the wall's 10 m thickness. A softer penalty cannot win that trade without also pushing the pillar route round the whole field.
2. The open width across a step from running counts of walkable cells along each line. The flat tops of blocks are walkable by the mask (only their edge ring is blocked), so a wall's own top counted as open ground. The open width now counts only cells of the field's region.
3. One cross-section per step. A diagonal step through the 6 m gap saw past the thin wall to the open ground beyond, so the route zig-zagged through the gap on diagonals. The open width is now the narrowest of three sections.
4. The squeeze alone. The wide-gap route still kept to the gap's edge: by flow time alone hugging is cheaper. The room term (1 + 2 x room given up) centres it; the crossing is now at x -292, 13 m from blocked ground.
5. The cost table built inside each plan under a lock: the first nine plans of the all-cases scene each waited about 40 ms, a 50 ms job. Tables are now built on the pool before the plans (four step classes in parallel); the first 16 of the 200-unit test took 13 ms with the table, the rest 2 to 3.5 ms.
6. Men aiming at the far route point and stepping round the wall in a fan of directions. Three men at the narrow gap's wall swapped between left and right every few ticks and never got through: the target was 67 m away, so the sideways pull toward the gap was tiny and the side flipped with each small move. Following the route line from the man's own place fixed it: near the gap, the point he can reach is right in front of it.
7. Dropping the route when the order clears (the centre within 18 m of the goal). Five men still behind the cube lost the route and stood at its wall. The route now stays while the unit holds at its end.
8. No reform on arrival. Men who came round an obstacle on the far side of the block stood 5 to 9 m from their slots, held up by comrades already in the ranks (the wide gap ran 82 s against a 66 s bound). Re-sorting the slots by where the men stand when the unit arrives brought it to 36 s.
9. Moving the frame on when the way to the next waypoint was clear of blocked ground only. On the river map that skipped the bridge waypoints at once (water is not blocked), and the unit waded beside the bridge. The skip now needs the way dry too.
10. The scene's orders and the arrival reform made after the sim step. Each requests a slot rewrite that the pipelined tick reads one job later than the inline tick, so `FL_PIPELINE=0` parted from the pipelined run from tick 120. The orders now go out at the head of the fixed tick, where a click's land, and the arrival re-sorts the slots in place between the install and the kick.
## Results per case

The final committed code (binary md5 cd70b35c), `FL_TEST_PATH=all` on the obstacle map, one regiment of 200 men-at-arms per case (21 files, 28 m wide, pace 6.0 m/s), ordered at tick 90 through `order_regiments`. Arrived = at least 99% of the men within 5 m of their slots; the bound is twice the route's length at the men's pace. Stuck = slower than 0.1 m/s for 10 s while more than 5 m from the slot. Log: `tmp/runs/path/final-all.log` (and `final-river.log`).

| Case | Route against straight | Waypoints | Planned (ms, cells) | 99% arrived | Bound | Most stuck at once | Verdict |
|---|---|---|---|---|---|---|---|
| cube | 256 m / 220 m | 4 | 1.2, 1,910 | 47.4 s | 85.6 s | 0 | OK |
| wide gap | 197 m / 120 m | 7 | 2.0, 2,949 | 36.0 s | 66.1 s | 0 | OK |
| narrow gap | 120 m / 120 m | 1 | 1.6, 1,606 | 29.2 s | 39.6 s | 0 | OK |
| pocket | 307 m / 225 m | 7 | 4.0, 9,034 | 56.0 s | 102.2 s | 0 | OK |
| zig-zag | 917 m / 244 m | 23 | 13.3, 80,406 | 153.5 s | 302.8 s | 0 | OK |
| closed box | 52 m to (40, 182), goal (40, 240) out of reach | 1 | 0.9, 27 | 11.6 s | 17.3 s | 0 | OK, stops |
| pillars | 242 m / 225 m | 11 | 2.5, 8,209 | 54.6 s | 80.4 s | 0 | OK |
| shared gap, from the north | 120 m / 120 m | 1 | 1.2, 520 | 104.6 s | 40.0 s | 57 | arrives, slow |
| shared gap, from the south | 120 m / 120 m | 1 | 1.5, 599 | 105.8 s | 39.9 s | 44 | arrives, slow |
| river, 15 m from the bridge's line | 126 m / 120 m, by the bridge | 3 | 0.7 | 56.6 s | 42.2 s | 0 | arrives, slow |
| river, far from the bridge | 120 m / 120 m, by the ford | 1 | 0.7 | 30.9 s | 40.2 s | 0 | OK |

- Every regiment arrives (9/9 and 2/2). Seven of the eight obstacle cases are inside their bound.
- The closed box: the order becomes a move to (40, 182), the nearest reachable ground with 14 m clearance, 58 m from the goal outside the box's south wall; the log has "the goal (40, 240) cannot be reached; going to (40, 182) instead". After arriving its centre moves 3.6 m in the first 6 s (the men settling into the re-sorted ranks) and then not at all for the rest of the run (logged every 10 s): no walking back and forth.
- The narrow gap and the pillars: the men stream through in files and the regiment forms up on the far side; 100% within 5 m of their slots after the verdict.
- The shared gap: both regiments get through and neither waits forever, but they stand head-on in the gap from about 19 s to 85 s (open question 1).
- The river: the regiment level with the bridge takes it (the frame goes up onto the deck at z 126 and down again), the far one wades; the log line names the crossing. The bridge regiment is slow because its wing men wade beside the deck (found on the way, below).
- Planning per path: 0.6 to 4 ms, the zig-zag 13 ms. The first batch of the scene waits one tick for its cost table (21 ms in that tick job), then the nine plans take 12 ms together.

## 200 regiments in one tick

`FL_TEST_PATH=many`: 200 regiments of 40 men in four rows south of the long wall, each ordered 150 m north through the wall's gaps in the same tick, maximized 2K window. 192 get routes and 8 (in front of the wide gap) walk straight. The first tick builds the cost table (7.9 ms), then 12 ticks of 16 plans (1.5 to 3.4 ms a batch); no regiment waits for its plan 0.47 s after the order. The longest frame from the order on: 8.0 ms in one run, 9.8 ms in the other (the limit was 20 ms).

## Determinism and main

Fingerprints every 60 ticks (`FL_HASH=60`).

- The all-cases scene with the final code (binary md5 29c63bd4, `target/flanks-final`, 130 s) run twice: 63/63 equal. Earlier builds: 61/61 and 63/63.
- The same scene with `FL_PIPELINE=0` against the pipelined run, final code: 63/63 equal. It took one fix. The scene's orders and the arrival reform first asked for a slot rewrite after the sim step, which the pipelined tick reads one job later than the inline one (the reform pass runs before the next step). Moving the orders to the tick's head made it worse (main's pipelined kick at tick t lines up with the inline prep at t + 1, so anything set before the step is seen a tick early inline). The fix writes those slots at once, between the install and the kick, where both ticks read them in the same job.
- `main` 66044d1 against the branch (frozen copy of a7d2f1b + 378bb61, `target/flanks-path`), the four scenarios of `work/scripts/gate.sh` (dir 40 s, archery 40 s, pile-wide 60 s, pile-two 60 s) on three maps, `FL_MAP` set: every one equal.

| Map | dir | arch | pilewide | pile2 |
|---|---|---|---|---|
| Grassland | 18/18 | 18/18 | 28/28 | 28/28 |
| Big Grassland | 18/18 | 18/18 | 28/28 | 28/28 |
| River | 18/18 | 18/18 | 28/28 | 28/28 |

On River the grid is built (it has risers and water), but no order in these scenarios crosses either, so no plan was made and nothing changed. The commits after that copy touch only route arrivals, the scenes and the table timing, none of which runs without a route.

Hash logs: `tmp/runs/scripts/gates/pf-{main,path}-{grassland,big_grassland,river}-*.hash`; the scene's `tmp/runs/path/hash-all-*.hash`.

## Cost

The 200k AI battle on Big Grassland (`bench4.sh`: 900 m locked camera, `FL_HEAVY_FRAC=0.4`, AI on, 80 s, engaged window from 30 s, `tickstats.py`), maximized 2K window (`FL_WINDOW=2560x1360`, `FL_UNITS=100000`), instrumented builds of main 66044d1 (md5 67f3adb0) and the branch 9736e92 (md5 a8e521d3), run main, branch, branch, main. Load 3.8 to 10.5 (one reading at the start of the last run); Gota's two viewskater windows open on the GPU throughout; GPU clock idle between runs.

| Run | fps mean / p50 / p10 | kernel ms mean / p50 / p90 | grid ms | field ms | kernel cycles per soldier | fixed tick p50 |
|---|---|---|---|---|---|---|
| main 1 | 206 / 205 / 198 | 9.12 / 9.11 / 9.67 | 2.86 | 2.25 | 2032 | 1.7 |
| branch 1 | 206 / 205 / 202 | 9.23 / 9.21 / 9.79 | 2.87 | 2.23 | 2059 | 1.7 |
| branch 2 | 206 / 207 / 198 | 9.18 / 9.15 / 9.72 | 2.84 | 2.28 | 2045 | 1.7 |
| main 2 | 205 / 204 / 201 | 9.24 / 9.21 / 9.85 | 2.88 | 2.30 | 2058 | 1.7 |

The same per tick within the run-to-run spread of main against itself (kernel 9.12 to 9.24 ms). Big Grassland builds no grid; the kernel pays one empty-route check per man.

On the obstacle map:

- The grid build: about 20 ms on the main thread at the battle's first tick (once).
- A cost table: 15 ms in a tick job of its own, on the pool, once per regiment width.
- Plans: 0.6 to 3.5 ms each, the zig-zag 10 to 14 ms (80k cells searched). 16 per tick at most, in parallel.
- 200 regiments of 40 men ordered in one tick: see the section above; the longest frame 8.0 and 9.8 ms in two runs.
## Pictures

`tmp/shots/pathfinding/<case>_<seconds>s.png`, taken with `work/scripts/shots.sh` and `FL_NAV_VIZ=1`, camera locked above each case at yaw 0, which looks south: north is at the bottom of each picture, east on the right, the start at the top. Red outlines are the blocked cells' edges, the yellow line the route (the legs behind faded), the circle the current waypoint, the cyan line the regiment's centre to it.

- `cube_25s.png`, `cube_40s.png`: round the cube's west side; the waypoint at its north-west corner.
- `wide_20s.png`: through the middle of the 42 m gap.
- `narrow_20s.png`: the men in a file through the 6 m gap, spreading again beyond it.
- `pocket_25s.png`: round the U's west arm, not into it.
- `pillars_25s.png`: weaving between the staggered posts.
- `zigzag_80s.png`: in the second corridor, the route through all three turns.
- `box_30s.png`: holding outside the closed box, its goal inside.
- `shared_45s.png`: the two regiments packed in the gap.
- `many_20s.png`: 200 regiments at the wall's gaps.
- `river_25s.png`: over the bridge, wing men wading beside it.

## Open questions for Gota

### 1. Two units passing each other in a gap (proposal, not built)

The shared gap gets both units through, but in 113 s against a 40 s bound. From 19 s to 85 s the two blocks stand head-on in the 14 m gap with up to 57 men stuck, then it clears and both pour through in under 30 s. Nothing in the route causes it: two crowds walking straight into each other press until the crowd yield (`yield_to_crowd`) zeroes everyone's drive, and the mass only frees by drift.

What a man does in a crowd coming the other way: he steps aside before he meets the oncoming man, by habit to his right, and the next man follows him; two lanes form. The proposal is that rule for each man: in the neighbour scan he already runs, a comrade ahead of him (within a few metres, inside a cone of about 30 degrees of his way) walking toward him adds a step to his own right to his drive, stronger the closer the man. Nothing else changes: he sees the man, he steps aside. Lanes are the known result of this rule in crowd models.

Risks of a worse feel: it acts wherever friendly units cross, not only in gaps (two units marching through each other on open ground would now part into lanes instead of shoving); men in a block whose rear meets its own front after a turn could sidestep each other. It costs a dot product per comrade in the scan for men who are walking. It is feel-critical, so it waits for your word; a switch to try it in play would come first.

### 2. How a wide formation goes through a gap or round a corner (proposal)

The plain version translates the block's slots along the route and lets the men find their way where their slots hit an obstacle. Seen in the pictures: the block keeps facing the final goal for the whole route (on the zig-zag it walks sideways for most of 900 m), and at a gap the men funnel in from both wings and re-form beyond it, which reads as a crowd, not a unit.

What I would change, in this order:

- Face along the leg. While a unit walks a route its formation faces along its current leg, so the front rank leads round corners, and turns to the ordered facing on the last leg. Each leg change is a reform (slots re-sorted by where the men stand), which looks like a wheel done by the men.
- Narrow to a column at a bottleneck. The planner already knows the open width on every cell (the cost table). The route can carry the narrowest open width of each leg; a unit whose leg is narrower than its front re-forms to as many files as fit (open width over the file pitch), and widens back to its own files on the next leg.
- Wheel at corners only when the corner is open ground; at walls the column does it.

### 3. Pathing on for every map

It is on everywhere now; `FL_PATH=0` turns it off. Maps with no blocked ground and no water build no grid and plan nothing, so Grassland and Big Grassland run as on main.

### 4. Found on the way (not changed)

- The river's bridge deck: a man wading next to the deck (within a metre of its south or north edge) reads the deck's 1.5 m lift in `slope_at` as a near-vertical slope and crawls at 0.1 to 0.2 m/s. On main the same happens to any man wading along the deck. In the river scene this is why the unit that takes the bridge arrives in 56 s against its 42 s bound: its wing men wade beside the deck. A fix belongs in the terrain (the slope a man feels should come from the ground he stands on, deck or bed, not across the deck's edge).
- Routed men flee in a straight line to their map edge and press into any wall in the way (out of scope: a path per soldier across the map).
- The grid build takes about 20 ms on the main thread at the first tick of a battle on the obstacle and river maps (once; a crater refreshes only round itself).
- The zig-zag plan is the slowest at about 10 ms (79k cells searched): the straight-line estimate is a poor guide in a maze. Plans run in the tick job after the soldiers, so at 200k it adds to a job that finishes well inside the 33 ms tick, but it is the case to watch if maps grow mazes.
