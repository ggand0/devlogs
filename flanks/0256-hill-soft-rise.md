# Soften the hill rise
Written by GPT-6 Astra.

2026-10-17. Gota found the 180 m climb better but still too steep and requested roughly another ten degrees of smoothing, with closer attention to the M2TW reference. Re-examined debug0.png and the actual in-game views hasting_m2tw0.png and hasting_m2tw3.png. They show a broad low shoulder, shallow humps and a gradual transition into the defender line. The previous crown remained too high relative to its climb depth.

Lowered the centre crown from 45 to 36 m and extended the main climb from 180 to 280 m, varying 262–298 m across the field. Big Hill doubles those lengths and heights. Widened the central 3 m hump's depth radius from 28 to 50 m to remove its concentrated extra grade. The integrated profile and the other humps remain.

Decoded heightmap maximum: 14.089 degrees, down 9.629 degrees; centre maximum: 13.936 degrees. Height PNG SHA-256 `381bb27f625a316488686bb26374f62025e9ab6b93761a5332266b1d852d9e94`, reproduced byte-for-byte. Maximum height quantization error: 0.275 mm. The approved ground texture PNG and KTX2 retain their prior hashes. Previous source assets and all earlier review artifacts remain intact.

All 49 tests pass, including updated profile and crown checks. Opt-dev build passes in 32.84 s; strict all-target clippy passes in 10.78 s. Logs: `tmp/runs/hill-soft-{tests,build,clippy}.log`. The diff changes the hill PNG, scale config, authoring script and hill tests only. Before-change Hill and Big Hill FL_TEST_DIR runs each saved seven fingerprints. Captures use note 094's helper and matching camera settings; the initial attacker and oblique views show the lower, gentler shoulder.

## Completed checks

All 49 tests, opt-dev build and strict all-target clippy pass. Hill and Big Hill simulation runs each differ in all seven fingerprint samples, first numerically at tick 60, as expected for changed heights. The existing hashcmp helper reports tick 120 first because it sorts tick strings lexically; the numeric comparison is saved beside the page. No game errors in the runs or captures. No additional Grassland control was run; this diff contains hill-specific assets and tests only.

Both 12-second orbit clips were inspected through sequential frames at 2 fps, including the Big Hill near-ground camera. No visible camera penetration or terrain holes. The HTML passes checks for five comparison views, six embedded figures, image enlargement, before/after buttons, wipe slider, video metadata and 800/390 px layouts. Results: `tmp/runs/hill-soft-browser.json`.

Saved as local commit `7086458` on `feat/hill-map`, not pushed. Height PNG tracked by LFS. The approved texture hashes are unchanged, and heightmap regeneration matches byte-for-byte. The pre-existing `target-merge/` directory remains untouched. The opt-dev build is ready for play review. Woods, track and dead tree remain a later stage.

2026-10-17 follow-up: Gota approved the softer hill shape ("ok now it looks good"). Terrain shape and ground texture are approved. Remaining visual work from handoff 120: woods and shoulder trees, limited hedges/scrub, the crest dead tree and dirt track, followed by planting counts and Big Hill performance/camera review. No planting started in this follow-up. Deployment, slope/fatigue, high-ground AI and the 200k benchmark move remain separate integration work.
