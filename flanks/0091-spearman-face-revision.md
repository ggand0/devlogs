# Spearman face revision

2026-09-23 — GPT-6 Astra.

Gota accepted the overall spearman and requested a more mature face, prioritizing the eyes and mouth. References were the generated men-at-arms sheet and M2TW armoured sergeants. Follow-up feedback identified oversized nostrils in the first pass and excessive nose-to-mouth spacing in the next. The final review corrects the nose's bridge/sidewalls, reduces its lower width, uses small nostril creases and restores compact spacing.

Work is isolated in ignored `assets_dev/spearman/textured_v2/`. Replaced the cylindrical face/separate wedge nose with a continuous 206-triangle face, recessed sockets, supporting lip/chin planes and explicit nasal bridge topology. Added original facial albedo with restrained eye whites, lids/brows, a closed mouth, stubble and weathering. A continuous front UV island now receives approximately 312 × 415 pixels within the existing 2048² atlas. The atlas was repacked and rebaked, with no added material or texture memory.

Final measurements: 2,998 L0 triangles (+37), 2,028 source vertices, 3,622 exported vertices, 1.80 m character and 2.52 m overall spear bounds. Part triangle coverage is body 0: 1,482; legs 2/3: 320 each; spear arm 4: 388; shield arm 5: 488. Source vertex coverage is body 1,026; legs 186 each; spear arm 221; shield arm 409. GLB pivots remain hips (±0.115, 0.910, 0), spear shoulder (-0.224, 1.435, 0), shield shoulder (0.224, 1.435, 0).

Independent GLB inspection and embedded-image/part checks pass: VEC4 COLOR_0, correct TEXCOORD_1 IDs/heights, one part per triangle, one opaque material and a matching embedded/external RGBA atlas. Source checks prove the other 23 geometry components and fourteen non-face texture files are identical to v1. SHA-256 checks preserve the recorded v1 inputs and committed knight asset.

Reviewed matching front, three-quarter, low and profile renders of the actual old/new GLBs, plus full-unit and 60/20/8/3-pixel L0 renders. Comparisons are in the revision directory. README contains reproduction commands and full counts; handoff is `tmp/drafts/handoff-spearman-face-v2-gpt6-astra.md`.

No engine/source changes, Cargo commands, live Blender access, commits, animation or LOD work. The face remains pending Gota's approval before the MAA variant. Full per-kind acceptance (authored lower LODs, silhouette comparison and rotation checks) remains pending.
