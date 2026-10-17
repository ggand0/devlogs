# Broaden the hill climb
Written by GPT-6 Astra.

2026-10-17. The 48 m upper face read as a cliff and was rejected. This revision spreads the rise over a nominal 180 m, varies that depth across the field, and replaces its constant slope and short joins with a continuously changing grade and broad rounded crown. Current work stays on feat/hill-map in flanks-gfx. Approved texture preserved.

The new nominal climb is 180 m (162–198 m across x), or 360 m on Big Hill. It integrates increasing grade over its first two-thirds and decreasing grade over its final third, rather than keeping a straight face. The crown stays at 45/90 m. Lower humps continue into the climb. The measured source-grid maximum is 23.718 degrees, centre maximum 22.110 degrees. This deliberately relaxes the 35–40 degree request because the resulting short face looked much steeper in play.

The height PNG reproduces byte-for-byte, SHA-256 `1668dd144e4419bf74b20abe9250fe32165c7e16a09fdb4a97fbf5fb6676c505`. The approved layout PNG and KTX2 keep their prior hashes. The old source PNG/script/config and complete prior review remain intact. Only the height PNG, authoring script and terrain tests change.

All 49 tests pass. Build completes in 36.80 s and strict all-target clippy in 11.31 s. Logs are `tmp/runs/hill-broad-{tests,build,clippy}.log`. Before-change fingerprints saved for Hill and Big Hill, seven each. Captures use `work/scripts/capture-hill-0093.py`, output `tmp/shots/hill-broad-climb-v2/`. The revised attacker and oblique views show a rounded shoulder and a longer visible slope. Review page generator: `work/scripts/viz/hill_climb_093.py`.

Capture launches encountered other games in the main tree. They were left untouched, and each capture waited for the guarded launcher. Process visibility in sandboxed reads was incomplete, so game guards and process checks were subsequently made through the host command environment.

Build, strict all-target clippy and all 49 tests pass. Both hill simulation checks differ in all seven samples, starting numerically at tick 60; this is expected because terrain heights affect movement. The hashcmp helper reports tick 120 first because it sorts tick strings lexically. Numeric results are saved beside the review in `hash-comparison.json`. No new Grassland control run was made: the diff changes only the hill asset, hill authoring script and hill tests.

Captured five before/after views and three additional near views, plus two 12-second orbit clips. Reviewed sequential orbit frames at 2 fps; no visible camera penetration or terrain holes. All game capture logs are free of errors. Headless Chrome verified five comparison pairs, all embedded images, before/after buttons, wipe slider, enlargement, video metadata and 800/390 px layouts. Results: `tmp/runs/hill-broad-browser.json`. HTML screenshot: `tmp/shots/hill-broad-climb-v2/review-page.png`.

Local commit `bed5a17` on `feat/hill-map`, not pushed. The heightmap is in LFS. No edits to the approved texture, camera, shaders, simulation code or main tree. The pre-existing `target-merge/` directory remains untouched. This revision is ready for play review; planting stays a later stage.
