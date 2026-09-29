# 0163: vegetation 200k measurement, fingerprints, text audit, PR draft

Written by Claude Opus 5.5.

2026-09-29, GPU free while Gota was away. feat/vegetation at 06146d5.

## Setup

- Before: e593c33 (three cascades, Grassland with the specimen row of eight plants), built in a detached worktree in the scratchpad (flanks-e593) with the graphics tree's target dir, copied to target/opt-dev/flanks-before. After: the planting (bd26ae0 code; 06146d5 only changes comments), copied to flanks-after.
- Trap: building a second tree into the same target dir left target/opt-dev/flanks as the other tree's binary; cargo saw the graphics crate as fresh and did not relink. A touch of src/vegetation.rs forced it. Verified each copy by strings: only flanks-before carries the scratchpad asset path, only flanks-after has "player rear copse" and "wood core".
- work/scripts/perf/veg-ab.sh: FL_TEST_FRONT=1 FL_UNITS=100000, AI on, locked cameras; flank (-470, 0, yaw -1.5708, pitch 0.35, 15 m) 120 s, edge (-440, -40, yaw 0.9, pitch 0.45, 60 m) and far (0, 0, pitch 1.2, 900 m) 75 s each; GPU clock sampled every 2 s; then gate.sh on both binaries. work/scripts/perf/veg_ab_summary.py averages the fps lines over the last 60 s (flank) or 40 s. Logs in tmp/runs/veg-ab/.
- Load average 2 to 11 during the runs (the game itself plus three docker containers: phpmyadmin, backend, db). GPU 1.85 to 1.92 GHz, 98% busy in the flank view.

## Results

| View | Before | After | Opaque pass | Shadow pass | update+post p50 |
|---|---|---|---|---|---|
| Flank | 107.6 fps (106 to 110) | 108.2 (106 to 111) | 1.05 / 0.96 ms | 0.40 / 0.40 | 2.99 / 3.06 |
| Edge | 168.1 (163 to 176) | 163.6 (156 to 168) | 0.55 / 0.58 | 0.24 / 0.26 | 2.78 / 2.83 |
| Far | 158.0 (144 to 164) | 153.8 (138 to 163) | 0.43 / 0.47 | 0.02 / 0.02 | 2.83 / 2.95 |

- Flank: no cost; the camera looks away from the west edge and the east trees are far cards.
- Edge and far: about 0.16 ms a frame. GPU share about 0.04 ms opaque and 0.02 ms shadow, far under the 1 ms target. The rest is main-thread work for about 900 plant entities (296 plants, parent plus parts): Bevy's per-entity visibility and extraction and `select_tree_level`, up to 0.12 ms in update+post in the far view.
- River (765 plants) not measured at 200k.

## Fingerprints

gate.sh on both binaries, prefix veg-flanks-before / veg-flanks-after: dir 18, arch 18, pilewide 28, pile2 28 fingerprints, byte-identical, no panics. The main5 baselines predate PR #11 and were not used.

## Text audit

- All 14 branch commit messages since main (Astra's and mine): no footers, no model or agent names, no "owner" or "Gota", no em dashes, no filler words; titles present-tense verbs.
- Every line the branch adds (against the merge base, not main's tip: main has moved to d4d375c and a two-dot diff mixes its lines in): same scan, clean. Two comment problems of my own fixed in 06146d5 "Fix the river planting and vegetation README docs": the river comment narrated the old box trees, and the vegetation README said the copses wait on collisions. The parking comment in `GRASSLAND_STANDS` stays; Gota asked for it.

## PR draft

work/drafts/pr-vegetation.md: "Add trees and shrubs to the Grassland and River maps".

## Before the PR

- main has PR #14 since the last merge. `git merge-tree` shows one conflict, src/game_state.rs: the Retarget scenario and Scene both joined the scenario table. Merge main into feat/vegetation and keep both, as in 0159.
- The branch adds 24 GLBs, 119.1 MB through LFS; 13 are loaded and 9 placed on a map. The branch is not pushed, so the unused ones could still stay out of the public LFS storage. Gota's call.
- Gota, 2026-09-29: the natural oaks (`oak`, `mature_oak`, `leaning_oak`, darker green, no suffix) are kept on purpose for future northern maps (a Sweden or Scotland look). Grassland moved to pale and lighter after they were made, judged without shadows. They are not leftovers; keep them in any trim of the shelf.
- Cleanup waiting on Gota's word: the scratchpad worktree flanks-e593 and target/opt-dev/flanks-before and flanks-after in the graphics tree (164 MB each).

## Against main (the comparison that should have come first)

Gota: the branch's cost is the third cascade, so the A/B belongs against main; measuring the planting against e593c33 hid it (memory perf-baseline-is-main). No overhead view: only the line-end angle, from both ends.

- flanks-main: main at 0a41859 (the main this branch merged; PR #14 since then changes melee orders only), built in the scratchpad worktree flanks-main-base with the graphics tree's target dir, then the graphics binary relinked with a touch. work/scripts/perf/main-ab.sh, order main, branch, branch, main, 120 s each, mean of the last 60 s:

| View | main | branch | Shadow pass | update+post p50 | frame p50 without a tick |
|---|---|---|---|---|---|
| West end, looking east | 111.3 fps (109 to 113) | 108.1 (106 to 110) | 0.30 / 0.40 ms | 2.88 / 3.08 | 5.86 / 6.06 |
| East end, looking west | 107.6 (106 to 110) | 105.7 (103 to 108) | 0.30 / 0.40 | 2.91 / 3.13 | 6.26 / 6.29 |

- 0.17 to 0.27 ms a frame: +0.10 ms GPU shadow pass (the third cascade holds terrain and trees) and about +0.2 ms main thread (the extra view and the plant entities). GPU 1.83 to 1.85 GHz, load average 2 to 10.
- Fingerprints against main: gate.sh on flanks-main (prefix veg-flanks-main) equal the branch at every tick both reached, all four scenarios. Main has one sample more per scenario (19 / 29 against 18 / 28): it starts faster, the branch loads 13 tree models at startup, and gate.sh runs a fixed wall-clock time.
- This matches Gota's own observation from playing 200k battles with the fps counter (devlog 0160): 114 to 115 fps with two cascades and 110 to 113 with three, at the same spots. The controlled runs give the same few-fps drop (1.9 to 3.2 fps, 0.17 to 0.27 ms), in both directions along the line, and place it: the extra shadow view and its pass, not the trees.
- PR draft performance section rewritten against main.

## Vegetation shelf

For deciding which GLBs the public repo keeps: a scratch worktree at 06146d5 (flanks-shelf, never committed) loads all 24 files in assets/vegetation and lines them up on Sandbox at z = -330, one family per shot, yaw 0, scale 1, sun behind the camera. All 24 load. Each shot has a labelled copy (*_labeled.png, used by the page): every file's number and name under its trunk, placed by projecting its world position through the locked camera (orbit around the focus, 45 degree vertical field of view); the overview labels each family. work/scripts/capture-shelf-0037.py (`annotate` redoes only the labels and the page); page tmp/shots/0037_vegetation-shelf/index.html with the table of files, sizes and status (placed, loaded but placed nowhere, not loaded). 119 MB in all, 46.8 MB placed.

Leftovers, removed after the PR merges (Gota): scratch worktrees flanks-e593, flanks-main-base, flanks-shelf; target/opt-dev/flanks-before, -after, -main, -shelf in the graphics tree.

## Shelf trim, Gota's decision

Of the 24 GLBs, only the 9 used stay in git: the `_pale` and `_lighter` oaks, `silver_birch_warm`, `shrub_a`, `shrub_b_sandbox` (46.8 MB). The other 15 leave the repository to keep 0.2.0 small and are kept locally as byte-exact copies (SHA-256 equal to their LFS object ids at 06146d5), each folder with a README saying what every file is and how to bring it back:

- assets_dev/vegetation/shelf/northern/: `oak`, `mature_oak`, `leaning_oak` (natural, darker green) and `silver_birch` (cool grey-green), for northern maps.
- assets_dev/vegetation/shelf/dark-bark/: `mature_oak_trunk`, `oak_trunk`, `oak_trunk_light`, for other maps.
- assets_dev/vegetation/archive/: the four `_original` files and `shrub_b` to `shrub_b_v4`, for the record.

Backed up to /data/ggando/flanks/assets_dev/vegetation/shelf and archive; checksums of every file equal the local copies. `load_trees` drops the four entries that were loaded but placed nowhere (the natural oaks and `shrub_b_v4`), so a fresh clone warns about nothing missing. The assets README describes only the shipped files. The upright shrub's build script reads `shrub_b_sandbox.glb`, which stays.

The removed files stay in the branch's earlier commits, so pushing the branch as it is still uploads them to LFS storage once; the 0.2.0 checkout and a clone of main at 0.2.0 no longer contain them.

Committed as eab2bb9 "Remove the tree and shrub variants no map uses". Build, strict clippy and the five vegetation tests pass; the load list equals the shipped files. In game, each map loads with no missing-file warning and the same plants as before: Grassland 131 trees and 165 shrubs, River 352 and 413, Sandbox 29.

The shelf and archive moved from resources/vegetation/ to assets_dev/vegetation/ on Gota's word (resources/ holds his screenshots and references only). `*.glb` is ignored in assets_dev ("build output, stays on disk"), so the models stay local there; the /data copy moved to /data/ggando/flanks/assets_dev/vegetation/ and checksums match.

## Merge of main and the history rewrite

- Merged main d4d375c (PR #14) as b33cf38: one conflict, src/game_state.rs, both Retarget and Scene joined the scenario table; kept both (Retarget, then Scene), `SCENARIO_ENVS` 10 to 11. Build, strict clippy and all 8 tests passed. A fingerprint gate of main's tip against the merge was stopped by Gota after main's dir and arch runs.
- Gota chose to keep the 15 removed GLBs off GitHub entirely: LFS objects cannot be deleted from a GitHub repository once pushed, and the branch had never been pushed. Backups first: branch backup/vegetation-before-lfs-trim at b33cf38, and a verified complete-history bundle work/backups/vegetation-lfs-trim-2026-09-29/feat-vegetation-b33cf38.bundle (21 MB, copy in /data). The removed LFS objects themselves are the assets_dev copies (the bundle holds pointers only).
- `git filter-branch --original refs/original-vegetation-lfs-trim --index-filter 'git rm --cached ... 15 paths' -- feat/vegetation ^main` in the graphics tree: only the 18 branch commits rewritten, main's commits untouched; an older refs/original/ backup from another session was left alone.
- Checks: tip tree byte-identical to b33cf38; none of the 18 commits contains any of the 15 files; both merges keep main parents 0a41859 and d4d375c; Astra's tree clean at the new tip 166642e; `git lfs push --dry-run origin feat/vegetation` lists only the 9 kept models (the three lighter oaks in two versions, before and after the bark unification).
- Every branch hash changed; work/backups/vegetation-lfs-trim-2026-09-29/hash-map.md maps old to new (e593c33 to 14afd86, 038f7ec to 0929f16, 0689b61 to da6d74b, bd26ae0 to f46f320, 06146d5 to 0e829c8, eab2bb9 to b35963c, b33cf38 to 166642e). Hashes in earlier devlogs, notes and handoffs are the old ones.
- (Superseded below: the messages were reworded.) After the trim some messages named models their commit no longer carried: 5676059 "Add four grassland tree variants" adds the loader and README for oak, mature_oak, leaning_oak and silver_birch without the models; dc8f9ee names the natural oaks and cool birch; b35963c's body says fifteen files "leave the repository", which they never entered.

## Reword and second audit

- Backups before each step: backup/vegetation-before-reword (166642e) and backup/vegetation-before-merge-msg (5936017) and backup/vegetation-before-amend (81343b3), each with a verified bundle in work/backups/vegetation-lfs-trim-2026-09-29/ and /data; filter-branch backup refs under refs/original-vegetation-reword and refs/original-vegetation-merge-msg.
- `git filter-branch --msg-filter` keyed on the commit hash reworded eight messages to what each commit now carries: the tree loader (was "Add four grassland tree variants"), lighter oak foliage (was "natural and lighter oak palettes and cool green birch"), the warm birch body, detail by on-screen size (was "Add low spreading shrubs"), the Sandbox copse with the fuller low shrub (was "... refine the mature oak trunk"), the lighter oaks' bark, the pale oaks' body, and "Load only the trees and shrubs the maps place" (was "Remove the tree and shrub variants no map uses").
- The last merge message carried git's "# Conflicts: src/game_state.rs" lines (`git commit --no-edit` keeps them) and then a trailing blank line; stripped, then amended to the plain title.
- Second audit: all 19 messages and every comment line the branch adds against main (157), read in full. Fixed in c5f7cb7 "Fix the camera sweep and shrub build docs": a camera comment spoke of "review sweeps", and the shrub A build README told readers to run work/scripts/flanks-run.sh, which is not in the public repo. The rear-corner copse note stays as Gota asked.
- Final tip c5f7cb7; tree of cf9dc27 identical to the pre-reword tip; no branch commit carries any of the 15 files; the LFS dry run lists the 9 kept models only. work/backups/vegetation-lfs-trim-2026-09-29/hash-map.md maps the pre-trim hashes to the final ones (e593c33 817b478, 038f7ec deae269, 0689b61 dd1614c, bd26ae0 cab0464, b33cf38 cf9dc27).

## Merged

PR #15 merged 2026-09-29 as a864d7e on main (branch tip d7bde27 after two last cleanups: 595a265 moved the bridge's mesh builder into water.rs and fixed stale vegetation docs, d7bde27 renamed it FaceMesh and removed "soup" from the bridge and chunk-mesh comments). Local main fast-forwarded with `git fetch origin main:main`; no tree switched. Removed with Gota's word: the scratch worktree flanks-e593 and target/opt-dev/flanks-before and flanks-after in the graphics tree.
