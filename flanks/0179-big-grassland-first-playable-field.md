# Big Grassland first playable field
Written by GPT-6 Astra.

2026-10-03. Gota approved the proposal and both shared-source edits, requested feat/big-grassland, and asked to start execution. Created that branch from main 25ec337. The old resources symlink blocked checkout; preserved the link in work/backups/big-grassland-2026-10-03/resources-link without touching its target, then switched successfully.

Local commit 809ef2a adds the 2048 by 2048 m map, per-map grid dimensions, chunk replacement on map switches, dynamic selection-ring dimensions, and the large-map camera range. Changes are confined to terrain.rs, camera.rs, vegetation.rs, battle_setup.rs and selection_rings.rs. There are no new asset files and no sim edits. The first field uses the existing pasture artwork and materials, with initial low hills and no vegetation yet.

At 2 m cells: 1025 by 1025 vertices, 1024 chunks, 2,097,152 triangles. The open 1700 m square measures 3.247 to 6.824 m in height, maximum grade 0.755 percent. Big Grassland's zoom limit is 2800 m, its far plane 6000 m; the old maps retain 900 m and 1000 m. Picking reaches the new overview.

Offline opt-dev build and strict clippy pass; 35 tests pass. Four old-map FL_TEST_DIR runs match the untouched baseline, 13 of 13 fingerprints apiece, 52 total. Captured the 200k standing-army scene with FL_ARMY_GAP=500, a saved BigGrassland setup with units beyond the old bounds, the overview and west hill, and continuous zoom/orbit with repeated crater updates. Inspected the recorded frame sequence, without claiming a full-rate flicker review or a controlled performance comparison. Captures: tmp/shots/big-grassland-v1/.

Detailed result and next steps for Claude: work/notes/031-big-grassland-first-field-2026-10-03.md. This is the first playable stage, ready for review. Planting, landform refinement and controlled terrain/shadow measurements remain later stages. Normal army packing and the default gap are unchanged. Nothing pushed or merged, no game left running.
