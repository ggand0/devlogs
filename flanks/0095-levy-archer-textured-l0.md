# Levy archer textured L0

2026-09-23 — GPT-6 Astra.

Built a separate levy archer from the supplied reference and Claude's written brief in ignored `assets_dev/archer/textured_v1/`. Used the plain coif/lowered-hood/tunic version: patched wool tunic, hose, low shoes, strung self bow, left bracer, right-hip quiver and belt knife. No optional buckler or armor. The newer hip-quiver brief overrides the older back-quiver table; character scale remains the contract's 1.80 m. Reused the accepted face without another face revision.

The first render exposed upper-hose/tunic and sleeve/cuff/bracer overlaps. Fitted the hidden clothing layers, conformed the patch to the tunic, added a continuous coif edge and filled the neck. Six quivered arrows include three broadheads and three bodkins beneath their fletched tails. All geometry and textures are original project work.

Final export: 2,941 L0 triangles, 1,865 source vertices, 3,637 exported vertices, 1.80 m human/overall height; one opaque material and one embedded 2048² RGBA atlas. Part triangles: body 1,621; empty draw arm 208; legs 320 each; bow arm 472. Source vertices: body 1,107; draw arm 112; legs 186 each; bow arm 274. GLB pivots: shoulders (±0.205, 1.435, 0), hips (±0.115, 0.910, 0). Part IDs 0/1/2/3/6; the full bow/string belongs to arm_bow 6, hip quiver/arrows to body 0.

Raw GLB inspection and embedded-image/part checks pass: COLOR_0 VEC4, retained team alpha, correct TEXCOORD_1 and pivots, one full-weight part per source vertex, one part per exported triangle, embedded/external image equality and preserved RGB at zero alpha. Recorded prior unit inputs remain byte-identical. Reviewed actual GLB front/back/side/detail renders and 60/20/8/3-pixel checks.

README, validation and review sheets are beside the asset; Claude handoff is `tmp/drafts/handoff-levy-archer-gpt6-astra.md`. Stop for appearance review. Bow animation, string deformation, nocked arrow, additional variants and lower LODs were not attempted. The thin bow/string become subpixel at small sizes; final ranged-unit readability remains a lower-LOD task.

Claude's active integration edits were left alone. No source/renderer/Cargo-file edits, Cargo commands, live Blender access, commits or asset promotion.
