# 0101: Seal the knight visor recess

Written by GPT-6 Astra.

2026-09-23. Gota accepted the MAA and spearman faces, then reported seeing through the knight's eye slits. The prior helmet used two isolated single-sided dark quads, leaving culling and angled coverage problems.

Created `assets_dev/knight/textured_v3_visor/`. Removed those quads and joined a dark recess directly to the existing ocular rim, 6 mm behind the front. Shared vertices connect its walls to the main shell; the crown cap is welded to that shell, and a dark bottom cap closes the underside. The outer helmet shape, ornament geometry, shield, clothing, equipment, pivot/joint nodes and exported non-body position sets are preserved. Earlier versions and current shipped assets were retained.

Final L0: 2,768 triangles (+38), 1.80 m tall, 1,761 source / 3,780 exported vertices. Body 1,066 tris; weapon arm 272; each leg 416; shield arm 458; sword 140. One opaque 2048² RGBA atlas, original source textures, untinted interior. Full pivot coordinates and reproduction commands are in the folder README.

Independent GLB, channel, texture-payload and per-triangle part checks pass. The joined helmet shell/interior is a closed 208-triangle solid with no nonmanifold edges, winding errors or degenerates and positive volume. A first separate insert passed straight-on review but allowed grazing sightlines around its edge; joining the recess to the shell resolves the disconnected coverage instead of disabling culling. The ray check samples 54,300 inward directions: 53,548 hit the interior, 510 are occluded before the slit, and 242 enter and leave the curved mouth at grazing silhouette angles without entering the solid. No unexplained background rays remain. Bright-background front, oblique and low-angle GLB renders were inspected with culling enabled.

Other pre-existing open boundaries remain: 490 non-shield edges and 12 winding inconsistencies outside the closed shell/interior. Full-model and authored lower-LOD acceptance are not claimed. The shield closure checks still pass. All 237 recorded prior files are byte-identical.

Updated `work/handoffs/handoff-faces-and-shield-backs-v3-gpt6-astra.md` with the knight path and counts. No src/, renderer, Cargo or animation work. No production asset replacement or commit by this task. Tracked asset modifications present at the start no longer appeared in git status at the end; their content hashes remained unchanged.
