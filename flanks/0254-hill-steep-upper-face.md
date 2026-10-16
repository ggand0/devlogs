# Give the hill a steep upper face
Written by GPT-6 Astra.

2026-10-16. Gota supplied `refs/map/hastings/debug0.png`, outlining the final rise below the defender line, and explicitly asked for that part to be 35–40 degrees. Continued on `feat/hill-map` in `flanks-gfx`. The approved ground texture remains unchanged.

The new profile keeps the crest at z = −160 m, raises the centre crown from 36 to 45 m, and concentrates the upper rise into the final 48 m of horizontal approach. Its middle 32 m is 37.5 degrees, joined to the lower approach and crown by 8 m smooth transitions. The centre face begins at z = −112 m and height 14.146 m, rising 30.854 m to the crown. Big Hill doubles all lengths and heights, preserving the angles: 96 m upper section, 64 m middle face, 16 m joins and a 90 m crown.

The lower grade is solved from the remaining height and length rather than multiplying the old incline. The existing three low rolls and depression remain, fading out before the steep face. The rear crown keeps its gentle fall. After PNG quantization, the middle face measures 37.494–37.801 degrees across the usable width, including its lateral slope. The sampled lower approach has median 1.677 degrees, 95th percentile 3.170 degrees and maximum 8.583 degrees. Whole-map maximum is 37.801 degrees.

The source script, PNG and height config from the previous ground are preserved under `work/notes/092-hill-upper-face-2026-10-16/previous/`. Current PNG SHA-256: `d27f2b1dc1458c08e0bf6fd7a7de2c485cc2147e8913015c4cd45263bc47840e`. Maximum quantization error: 0.343 mm on Hill. `hill.json` now supplies 45 m vertical scale; the same file-backed loader and 2 m runtime grids remain.

Before mutation, saved FL_TEST_DIR baselines for Hill, Big Hill and the unchanged Grassland control, seven fingerprints each. Changes affect terrain inputs to the sim, so mismatched hill hashes are intentional and must be reported. After-edit results follow below.

All 49 tests pass, including new game-loaded checks of the face angles on both sizes and retained camera clearance checks. Opt-dev build and strict all-target clippy pass. Only `assets/terrain/prepare_hill.py`, `hill_height.png`, `hill.json` and the hill terrain tests in `src/terrain.rs` change. No camera, shader, material or shared source edits. Logs: `tmp/runs/hill-face-{tests,build,clippy}.log`.

The after-edit runs differ on both hill maps, 0/7 equal, first numerical mismatch at tick 60. Grassland is unchanged at 7/7 equal. These are intentional changes to hill simulation inputs. Exact comparisons are saved in note 092's `hash-comparison.json`; the existing hashcmp helper orders tick strings lexically and reports tick 120 first.

The review HTML contains five before/after views, three additional near-face views, the measured profile and the marked reference. Two 12-second orbit clips cover the face and Big Hill minimum-zoom clearance. Inspected their sequential frames at 2 fps: no visible terrain holes or camera penetration. The runtime camera clearance sweep retains at least 1.500 m on both hill sizes. No errors in capture logs. Headless Chrome checks pass for all five image pairs, embedded images, comparison buttons, wipe slider, image enlargement, video metadata and 800/390 px widths. Browser results: `tmp/runs/hill-face-browser.json`.

Committed the four terrain files as `cd0dce2` on `feat/hill-map`, not pushed. The height PNG is tracked by LFS. The approved ground PNG and KTX2 remain byte-identical. The existing untracked `target-merge/` directory is untouched. Planting remains a later stage after this shape review.
