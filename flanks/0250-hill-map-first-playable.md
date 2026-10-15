# Hill map first playable terrain
Written by GPT-6 Astra.

2026-10-15. Gota approved beginning implementation after the visual layout review, including stronger 2–4 m broad rolls and a less even crest to match the attacker-side M2TW reference. Created `feat/hill-map` from main ab80162 in `flanks-gfx`. The existing untracked `target-merge/` is untouched. Scope for this stage: file-backed Hill/BigHill heights, menu selection, existing pasture materials and an in-game shape review before vegetation and track work. No shared source file edits authorised or planned.

Baseline build and old-map hashes run before editing the terrain code. Results and first playable review follow below.

## First playable ground

Committed as `1fd2078 Add file-backed hill maps`. Six files only: `src/terrain.rs`, `src/vegetation.rs`, `src/camera.rs`, and `assets/terrain/{hill.json,hill_height.png,prepare_hill.py}`. No push or merge. The height PNG is stored through Git LFS; its existing terrain PNG pattern already applied.

Hill and Big Hill are in the menu and load through `FL_MAP=hill` / `FL_MAP=big_hill`. Hill is 1024 × 768 m and Big Hill 2048 × 1536 m. A shared 513 × 385 unsigned 16-bit greyscale PNG supplies the heights. Hill uses the source grid directly; Big Hill interpolates it into 1025 × 769 vertices, doubling the ground and height. Both runtime grids retain 2 m cells. `hill.json` contains the one tunable height scale, initially 36 m. The loader uses `game_files::read`, so it follows the existing embedded-assets path too.

The main climb rises 36 m over 440 m on Hill and 72 m over 880 m on Big Hill, an 8.18% average. Centre crest z is −160 / −320 m. The crest bends by up to 14.39 m in z on Hill and has unequal height along x. The first engine images showed the broad 2–4 m rolls too weakly. Their lengths were reduced, preserving the first source and PNG in the note's `shape-v1/` folder. The final local maximum grade is 20.88%, with an 8.76% median and 11.24% 95th percentile in the sampled approach. The source recreates the final PNG byte for byte, SHA-256 `13d9348ccda25702d0b08c1e9682cd5e062592980d0927c563f6bd1f18a74c8f`. Quantization error stays below 0.275 mm on Hill.

This remains a bare-ground review using Grassland's current pasture material and colour layout. Trees and shrubs are both zero. Woods, track, bare-tree landmark and dedicated ground layout follow the terrain review. No performance comparison is claimed yet. The frontal view reads more quietly than the wooded M2TW reference; the oblique screenshot and orbit show the raised crown more clearly.

## Verification

The initial FL_TEST_FRONT baselines were retained as diagnostics, but inspection found its commands depend on frame timing. Replaced the preservation check with FL_TEST_DIR, matching the earlier Big Grassland process. Before/after runs on Grassland, Big Grassland, Classic, River, Sandbox, Obstacles and Town each have seven matching hashes at ticks 60 through 420: 49/49 equal. Files are `tmp/runs/scripts/hill_dir_{before,after}_<map>.{hash,log}`. No panic or wgpu errors. The subsequent camera-only change does not modify simulation inputs.

The stronger humps exposed insufficient camera clearance at distance 15 m and pitch 0.25 rad: the old transform came within 0.745 m of Big Hill. Added a general camera-eye height check against the terrain beneath it, with 1.5 m clearance. It retains the focus and leaves ordinary orbit positions unchanged when clear. Sampled the minimum-zoom/pitch camera on a 16 m grid at 16 yaw angles over every menu map; the final minimum is at least 1.5 m. A separate 12-second near-ground orbit on Big Hill records the western hump in motion.

All 48 tests pass. Tests cover 16-bit decoding, axes, bilinear sampling, scale validity, exact doubled height samples/grades, map dimensions, picking at far corners, switching/craters and camera clearance. Final opt-dev build and `cargo clippy --profile opt-dev --all-targets -- -D warnings` pass. Logs: `tmp/runs/hill-map-{tests,build,clippy}-final.log`. `git diff --check` is clean.

All game launches, builds and clippy runs followed `flanks-run.sh`. A different game appeared briefly during the initial baseline work; no process belonging to another person was stopped. No game is intentionally left running.

## Review and handoff

[Note 090](../work/notes/090-hill-first-field-2026-10-15.md) and its [HTML review](../work/notes/090-hill-first-field-2026-10-15/index.html) contain actual engine captures for Hill with 40k soldiers and Big Hill with the full 200k, the M2TW comparison, decoded height contours/profiles and a 12-second orbit. The private setup JSON files put the defending blocks on the crest and the attacking blocks in the valley using the existing setup format. The production spawn layout, deployment zones and slope rules remain for the Claude branch in handoff 120.

Inspected the sequential orbit frames: the terrain remains continuous through the turn, including the low-camera pass. This is a visual continuity check, not a frame-pacing or performance result. CLI/menu captures confirm both new Map buttons and selected variants. The HTML has nine embedded images, click enlargement, and an adjacent retained video file. Browser checks cover image decoding, video metadata, enlargement/close and 800/390 px layouts. Added a narrow-screen table layout and checked that the page fits the viewport, allowing for the browser scrollbar.

Capture tool: `work/scripts/capture-hill-0090.py`. Review generator: `work/scripts/viz/hill_field_090.py`. Capture logs and first versions are retained in `tmp/shots/hill-first-field-v{1,2}/`. Current images and both videos are retained under note 090. [Handoff 121](../work/handoffs/121-hill-first-ground-2026-10-15.md) records the ground format and the next work. Stop here for the ground review before planting.
