# Knight L2 distant model

Written by GPT-6 Astra.

Created the first step from `tmp/handoffs/HANDOFF-unit-lods-for-astra-2026-09-24.md`: knight L2 only, held for Gota's review. The candidate is `assets_dev/knight/lod_l2_v1/knight.glb`, with the original 2768-triangle L0 and a 250-triangle L2. The build script is also copied to `tools/blender/knight/build_knight_l2.py`; it requires the existing visor authoring source folder. Nothing was installed or committed by this task.

L2 is authored from closed low-polygon shapes. It keeps the source height of 1.80 m, all six used part IDs and their pivot heights, the heater shield and sword. The shield has a 23 mm closed board, separate wood back and leather rim atlas samples, plus two closed straps. The source atlas supplies all far colours, including team alpha. Uniform samples over each low triangle select an existing atlas texel matching its local mean linear colour and mask. The GLB is assembled by appending the exported L2 geometry to the original binary buffer; L0 and all existing node, texture, material and accessor data are preserved exactly.

L2 triangles: body 72, sword arm 20, left leg 28, right leg 28, shield arm including shield 72, sword 30. The first draft was 238 triangles; silhouette checks exposed undersized arms, feet and blade, so the final allocation spends 250 triangles on those shapes. L2 has zero open, nonmanifold, inconsistent-winding edges or degenerate triangles. The untouched L0 still has its documented 490 open edges and 12 inconsistent-winding edges, so this is not full-kind acceptance.

At 50 degrees, silhouette overlap is 90.58% front, 90.76% back, 89.74% sword side. Projected area is 99.19%, 96.33% and 98.89% of L0 respectively. Eight-view exported-atlas measurements give linear RGB (0.23564, 0.23240, 0.22021) for L0 and (0.23396, 0.23025, 0.21838) for L2, all channel differences below 1%. Visible team share is 33.41% versus 34.32%. Rendered comparisons cover 20, 8 and 3 px in red and blue. A 0–60 degree rigid-joint sequence checks the legs, arms and separate grip pivot; both arms and legs retain torso overlap at every sampled step. These are asset checks, not game animation acceptance.

Extended `tools/blender/inspect_surfaces.py` to inspect each mesh and combine primitive seams within each mesh. Preserved its original single-mesh return shape and `load()` API for existing callers. Regression checks compare L0 inside/outside the candidate and across a fixture split into two primitives. Concurrent work committed that inspector in `5f00e03` while this task was running; its changes are already in HEAD. No commits or staging were performed here.

Review images, joint animation and validation scripts are under `assets_dev/knight/lod_l2_v1/`. Detailed numbers, limitations and rebuild commands are in `tmp/notes/knight-l2-for-claude-2026-09-24.md`. Stop here for review; knight L1/L3 and the remaining kinds are not started.
