# Plant the hill woods and shrubs
Written by GPT-6 Astra.

2026-10-17. Gota approved the softer terrain and requested the trees and shrubs from the original layout, explicitly excluding the dead tree and dirt track. Work continues in flanks-gfx on feat/hill-map. The approved terrain and texture stay unchanged.

Implemented nine planting groups in src/vegetation.rs: three rear woods, two shoulders, two outlier groups and two sparse shrub strips. Counts match the planned targets exactly: Hill 480 trees / 180 shrubs; Big Hill 1600 / 500. Hill mixes 393 oaks and 87 birches, with 38 shrub A and 142 low B. Big Hill mixes 1295 oaks and 305 birches, with 90 A and 410 B.

Stand boundaries scale with the large map; plant size and cluster spacing remain in metres. The entire x = −440..440, z = −286..364 corridor on Hill stays clear of crowns at each asset's largest scale, doubled on Big Hill. The rear woodland edge undulates around z = −318. The existing Grassland generator is shared without changing its draw sequence, definitions or clearance. New Hill draws use a separate vegetation-only seed range. No simulation inputs or random streams change and no hash check is needed.

All 50 tests pass, including a new test that reads the shipped plant GLBs and checks exact counts, full crown clearance, field bounds, physical size and reproducibility for both hill sizes. Build passes in 33.12 s, strict clippy in 11.06 s. Only src/vegetation.rs changes. The approved heightmap and ground texture are unchanged.

Captured matching before/after views against note 094, new woodland/shoulder/shrub close views, and two 12-second orbit clips. Inspected sequential orbit frames at 2 fps: no visible floating roots or terrain penetration; existing detail levels change as trees pass the camera. The planting diagram uses actual runtime debug positions, saved to note 095's planting.json. No game errors in capture logs. HTML controls, all images, video metadata and 800/390 px layouts pass headless browser checks, saved in tmp/runs/hill-planting-browser.json.

A main-tree game was left untouched while captures waited. Performance uses the retained before-planting binary and the current binary, with 200k active FL_TEST_FRONT battles, 2560x1360, locked flank camera, sampled focus, GPU clocks/utilization and system load. No synthetic desktop input is used.

## Review result and performance status

Runtime planting matches every requested stand count. Hill has 393 oaks and 87 birches, plus 38 shrub A and 142 low B. Big Hill has 1295 oaks and 305 birches, plus 90 A and 410 B. The placement JSON and diagram come from the actual runtime debug logs. All 13 captures completed without game errors. Both 12-second clips were inspected through sequential frames at 2 fps; no visible floating roots or terrain penetration. The existing tree detail levels change as the orbit brings trees closer. Browser checks pass for all comparison views, embedded images, controls, video metadata and 800/390 px layouts.

The requested performance environment was FL_WINDOW=2560x1360, FL_TEST_FRONT=1, FL_UNITS=100000, locked west flank camera at x=−982, z=0, yaw=−π/2, pitch=0.35, distance=15. Each run lasted 70 seconds, excluding the first 20 seconds and samples after 65 seconds. Focus, actual window size, system load and GPU clocks/utilization were sampled each second. The window manager constrained the actual client size to 2494x1360 in all three runs.

The before-planting Big Hill run supplied 23 focused FPS samples: mean 170.78 fps, mean reported frame time 5.860 ms, GPU clock median 1890 MHz, GPU utilization mean 91.4%, one-minute system load 3.82–4.41. The planted Big Hill and Big Grassland runs each lost focus before the measured interval, so they supply no valid focused comparison samples. System load also varied, reaching 11.61 in the planted run. No vegetation-cost result is claimed. The focused comparison remains incomplete; Gota was asked whether to leave the next benchmark window focused or defer this check. No synthetic input was sent. Full observations are retained in `tmp/runs/hill-planting/` and `performance.json` beside the review.

Planting saved locally as `19246fa` on feat/hill-map, not pushed. Only src/vegetation.rs is committed. Approved terrain and textures remain unchanged; the existing untracked target-merge directory is untouched. The dead tree and dirt track are outside the requested scope. The current build is ready for visual/play review, with focused performance measurement still outstanding.

## Gota approval — 2026-10-17

Gota approved the current Hill and Big Hill trees and shrubs: “ok note in the devlog that I approved this. From here I'll clean up with Claude”. The terrain shape, ground texture and planting are now approved. The dead tree and dirt track remain excluded as requested.

Gota will continue cleanup with Claude. Work in this session stops here and the graphics files are released. The focused performance comparison remains incomplete as documented above and passes to Claude with the remaining cleanup; this approval does not establish a performance result.


Claude cleanup handoff: `work/handoffs/134-approved-hill-map-for-claude-2026-10-17.md`. It consolidates the approved terrain, texture and planting, branch state, validation, incomplete performance comparison and separate gameplay integration work. No source changes or additional runs were made for this handoff.
