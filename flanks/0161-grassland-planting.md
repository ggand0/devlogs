# 0161: grassland planting, first pass

Written by Claude Opus 5.5.

2026-09-29. From work/handoffs/070-grassland-vegetation-for-claude-2026-09-29.md (Astra's placement plan). Commit 038f7ec "Plant the grassland's margins, gap copses and hedge" on feat/vegetation, not pushed. Review page: tmp/shots/0035_grassland-planting/index.html.

## How Astra asked for it

Gota moved Grassland placement to Claude so Astra's time goes to asset authoring (0155, last section). Astra wrote the plan from its placement drawing (work/notes/023-vegetation-design-2026-09-27/composition.svg) and the accepted Sandbox composition:

- Assets: the Sandbox set only, reused through the existing handles and LOD path. Pale and lighter oaks in all three shapes, `silver_birch_warm`, `shrub_b_sandbox` as B (not v4, whose middle level is sparser), `shrub_a` sparingly. No GLB rebuilds, no palette multipliers.
- Mix: about 80% oaks and 20% birches; mature crowns anchor groups, upright oaks vary the skyline, leaning oaks soften edges; pale and lighter mixed without a pattern. B the majority of shrubs, A about 15 to 25%. Trees at scale 0.85 to 1.10, shrubs 0.75 to 1.20, yaw free.
- Exclusions: both deployment strips (player zone from `orders::deploy_zone`, the enemy's its mirror) with the real `FL_ARMY_GAP`, whole scaled and rotated crowns rather than trunks, leaning crowns included, far-card padding not counted; inside the field; the central 640 m of the gap clear.
- Regions: a broken western oak edge with openings and scrub (no avenue, no wall); a sparser, drier eastern edge with more leaning oaks; a few deeper copses at the outer ends of the gap near (-435, 0) and (430, 5); a short broken hedge where the eastern green/dry boundary crosses the gap, with an opening; a few low B groups in the 8 m rear margins, no framing of all four sides.
- Counts: 120 to 150 trees and 150 to 200 shrubs to start.
- Implementation: in src/vegetation.rs, the Grassland arm; Sandbox, River and Classic unchanged; deterministic vegetation-local seeds; no per-plant asset allocation; counts logged by species and region; a bounds check covering non-default gaps and the largest scales; no FL_ switch without its hash baseline.
- Review: stop after the first planting for Gota's look; plan with exclusions, a troop view for scale, both lighting directions, a zoom/orbit clip; performance after the review.

How each point was met:

| Request | Implementation |
|---|---|
| Sandbox assets, shared handles | Role tables `MATURE_OAKS`, `UPRIGHT_OAKS`, `LEANING_OAKS`, `BIRCHES`, `SHRUB_A`, `SHRUB_B`; `spawn_authored_plant` as Sandbox uses it |
| 80/20 mix, mature anchors | Clusters open on a mature oak; other trees birch 15 to 25% per stand, leaning 20 to 50%, mature 15%, the rest upright; result 42/34/29 oaks, 26 birches |
| A 15 to 25% | 0.24 in the side stands, 0.35 in the hedge; result 16% |
| Real gap, both zones, whole crowns | `Clearance` from `deploy_zone_for(terrain, army_gap())` and its mirror; crown reach from the L0/L1 meshes at the largest scale of the kind |
| Open centre 640 m | Third keep-out rectangle x -320..320 over the gap |
| Regions | 23 stands (below); hedge placed from the layout image's actual dry crossing at x 310 to 335 |
| Counts | 131 trees, 165 shrubs |
| Proof of clearance | Test at gaps 20, 60, 120 m with the real reaches |
| Logs | Per stand, per species, rejections by reason at info; every plant at debug |
| No sim input | Stateless `hash01` on vegetation seeds; plants stay scenery |

Gota's review, 2026-09-29: placement and appearance accepted ("It looks great"). His next wish from watching it: soldiers engaging near the edges of the open centre walk through the gap copses' trunks, so he wants simple tree and soldier collisions; see the end of this entry.

## What it does

The Grassland arm of `respawn_vegetation` no longer plants the specimen row. `grassland_planting` places the accepted Sandbox assets (pale and lighter oaks in three shapes, the warm birch, shrubs B `shrub_b_sandbox` and A) in 23 authored stands:

- West margin: six stretches (z -372..-300, -262..-205, -170..-70, 45..150, 190..250, 290..372) with openings between them.
- East margin: six shorter groups, more leaning oaks, fewer trees.
- Army gap: a copse at each outer end, (-435, 0) and (430, 5), plus a small outlier at (-362, -4).
- A short hedge in two runs at x ≈ 330 with an opening at z -5 to 7: the layout image's dry ground crosses the gap at x 310 to 335 (sampled red minus green along the gap), just east of the open centre's edge at 320.
- Rear margins: three small B shrub groups on each side, 8 m deep strips.

Inside a stand, trees come in clusters: four in five open on a mature oak at the centre with one to four more trees within 7 to 16 m, one in five is a lone tree. Shrubs come in groups of two to five, 70% of the groups at the crown edge of one of the stand's trees. Scales: trees 0.85 to 1.10, shrubs 0.75 to 1.20, yaw free.

## Clearance

- Each asset's crown reach is measured at load (`horizontal_reach`): the farthest horizontal distance of any L0 or L1 vertex from the trunk base, so the circle covers every yaw and the off-centre leaning crowns. The far card is left out (transparent padding). Mature oak 7.54 m, upright oak 5.84, leaning oak 4.35, birch 4.25, shrub B 1.36, shrub A 1.31.
- A candidate is turned down unless the circle of reach times the largest scale of its kind stays inside the field and clear of three rectangles: the player's deployment zone, its mirror for the enemy, and the open centre (x -320..320 over the whole gap).
- The zone comes from a new `orders::deploy_zone_for(terrain, gap)`, which `deploy_zone` now calls with `army_gap()`. One formula for deployment clamping and planting, so the two cannot drift, and the test can pass other gaps. This touches src/orders.rs, outside the graphics files: a visibility split with no behaviour change.
- `terrain::build_terrain` is `pub(crate)` so the test can build the Grassland heights.
- Spacing: trunk distance at least 0.55 (tree-tree), 0.5 (tree-shrub) or 0.75 (shrub-shrub) times the sum of the two crown radii at their scales. Slope limit 0.35; nothing on the grassland reached it.
- Draws: a counter stream through the stateless `hash01` on vegetation-only seeds. No sim random source, no FL_ switch, no terrain or sim input touched, so no hash run.

Tests (opt-dev): `grassland_crowns_keep_clear_of_deployment_and_the_open_centre` loads the shipped GLBs for their real reach and checks every plant at gaps 20, 60 and 120 m against the field and the three rectangles, at the largest scale, with its own clamp-to-rectangle distance. `grassland_planting_is_the_same_every_run`. Both pass, with the three existing mip tests. Build and strict all-target clippy clean.

## Result

Default gap: 131 trees (42 mature, 34 upright, 29 leaning, 26 birches = 20%) and 165 shrubs (139 B, 26 A = 16%); every stand full except the south hedge run (8 of 9). Candidates turned down: 597 for clearance, 562 for spacing, 0 for slope. Gap 20: 130 trees, 152 shrubs (the hedge mostly falls into the zones, 3 shrubs left). Gap 120: 131 and 165.

Tuning on the way: the first pass drew 50 mature oaks of 102 (anchors plus 40% of the rest); the rest share went to 15%. Shrub A came out at 14%; the side stands went from 0.20 to 0.24.

`RUST_LOG=info,flanks::vegetation=debug` prints every plant (name, position, yaw, scale, crown, stand); info prints per-stand and per-species counts and the rejections. work/scripts/viz/grassland_plan.py draws the plan from that log over the layout image.

## Captures

work/scripts/capture-grassland-0035.py (from Astra's 0034 script): overview, west edge lit and side-lit, west copse lit and against the sun, east copse with the hedge, east copse lit, east edge against the sun, hedge close, shrub group close, three 200k views with both armies standing (`FL_AUTOSTART=1 FL_DEPLOY=0 FL_AI=0`), and a 22 s zoom/orbit over the west copse. No panic or shader error in any log. The clip's log has 649 level switches (28 to L0, 293 to L1, 328 to the card).

## Open

- From a low angle along the margin the west edge reads as a line on the crest of the rising ground: the 30 m margin fits at most two staggered rows for a mature oak (8.3 m crown at its largest scale leaves a 16 m band). A deeper wood needs a wider scenery margin, a map and deployment decision.
- The hedge is two straight runs.
- From 900 m the west edge is a thin rim along the field edge.
- The natural `oak`, `mature_oak`, `leaning_oak` and `shrub_b_v4` still load at startup but no map places them now. Ask Gota whether to drop them from the load list.
- Map switching not driven (needs clicks on the Map row; no synthetic input). Same despawn-and-replant path Sandbox uses, with shared handles.
- Next per the handoff, after Gota's placement review: the populated 200k battle measurement, pre-placement branch (e593c33, with its specimen row) against 038f7ec, same three cascades, near, flank and far views, CPU, GPU, vegetation shadow and frame time, target under 1 ms added GPU. Then wind.

## Tree collisions, proposed, not started

Gota wants simple collisions between trunks and soldiers after seeing engagements at the edges of the open centre pass through the gap copses. Recommendation: its own branch off main after vegetation merges, not this one.

- It is a sim change on the default map: ask-first code, fingerprints, and a feel pass by Gota. The vegetation branch is render-only and close to merging.
- The trunk positions come from `grassland_planting`, whose crown reach is measured from the GLBs at load. The sim must not depend on render asset files: the reach needs a fixed table in the sim-side data (a test checks it against the GLBs), or the trees become map data, which is where 0.3.0's authored battlefield goes.
- There is no pathfinding. A regiment ordered into a copse will bunch behind trunks; how the formation and the melee footwork behave around them is feel-critical.
- Minimal version: trunks only (shrubs stay passable), each a static circle of the trunk-base radius that the soldiers' collision step keeps them out of, built once per map from the same deterministic placement. Gate: the four fingerprint scenarios fight in the centre, far from any trunk, and should stay identical; a new scenario marches a regiment through a copse. First look: how the sim treats the existing impassable mask (terraces, crater lips), which works per 2 m vertex, too coarse for a trunk.

## Gap copses parked in the rear corners until trunks collide

Gota's call after seeing flank fights at the edges of the open centre pass through the gap copses (resources/vegetation/debug17.png, his sketch on the plan): collisions go to the next branch, and until then the west gap copse, its outlier and the east gap copse stand in the rear corners of the deployment zones. The hedge stays: shrubs stay walk-through in the collision plan too.

- The three original stands stay in `GRASSLAND_STANDS` as commented lines that compile as they are; the comment above them says to restore them and drop the rear-corner stands with the collisions.
- New `Stand::inside: Option<Rect>`: a confined stand keeps its whole crown inside that rectangle instead of outside the zones. `PLAYER_REAR_CORNER` x -482..-335, z -376..-327; `ENEMY_REAR_CORNER` its mirror.
- Why those corners are free: at 200k the spawner lays 100 regiments of 1,000 (47 by 22 men at 1.4 m, 65.8 by 30.8 m blocks, 10 m apart) in ranks from the gap back. Gap 60: 8 ranks of 13, 13, ... and 9; the full ranks end at z 305.6, the partial rank spans |x| <= 330. Gap 20: 9 ranks of 12 and a last rank of 4; full ranks end at 326.4, the last spans |x| <= 146. Gap 120: 7 ranks of 15 and a last rank of 10; full ranks end at 294.8, the last spans |x| <= 322. So |x| >= 335 and |z| >= 327 holds no spawned block at any of these gaps. A player can still deploy into the corners by hand.
- The clearance test checks confined stands against their rectangle and every other plant as before.
- While waiting on a game another session had started (pid 2210556, flanks-audiolog in another session's scratchpad), a `;` in my command let a 9 s clippy run with that game up. Commands chain with `&&` after the guard from here on.
- Committed as 0689b61 "Move the gap copses to the rear corners until trees collide". Tests, strict clippy and build clean. Default gap: still 131 trees (44 mature, 40 upright, 26 leaning, 21 birches = 16%) and 165 shrubs (136 B, 29 A = 18%); the corner areas change the draws. Plan tmp/shots/0035_grassland-planting/plan-parked.png; both rear corners captured with 200k standing (armies-player-rear, armies-enemy-rear): the copses stand behind the last ranks, clear of every block. Review page updated.
