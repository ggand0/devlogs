# 0098: Spearman face and coif contour fit

Written by GPT-6 Astra.

2026-09-23. Gota identified beige skin outside the face/neck contour in `resources/maa_feedback0.png`, most evident from the front. The cause was the previous rectangular face-perimeter extension used to fill the mail-coif opening. Gota requested a spearman-only correction and review before transferring it to MAA.

Created `assets_dev/spearman/textured_v3_face_fit/`, preserving earlier versions. Removed the extension, restored the original anatomical jaw/cheek boundary, and fitted the coif opening as a curved outline against the face surface. Its edge samples the actual mesh depth and sits 2 mm forward to overlap. Added a closed neck behind the coif. The face albedo and its landmark projection are unchanged; no texture painting, alpha cutout or renderer change.

Final L0: 2,992 triangles (face 250, mail coif 192, new neck 20), human height 1.80 m, overall height 2.52 m including spear. Source/exported vertices 1,892/3,643. Same opaque material, embedded 2048² RGBA atlas, VEC4 color and UV1 part/pivot channels. Exact part coverage and pivot positions are in validation.json and README.

Viewed exported GLB front, 45°, profile and low-front renders with back-face culling. The rectangular beige strips are replaced by mail following the cheeks and jaw. Independent glb_inspect, texture payload/channel checks and topology checks pass for the changed closed face/neck and existing closed shield. Other clothing/trim open boundaries are inherited; full-model closure is not claimed.

Source comparison: only face and mail_coif changed, neck added; all 26 other geometry components match. All source PNGs match the preceding revision. The rig, motion source and runtime table are unchanged, so no animation was regenerated. All 354 recorded prior files remain byte-identical, including MAA and shipped assets. No game-code changes, Cargo runs, asset promotion or commits.

Review: `assets_dev/spearman/textured_v3_face_fit/face_fit_comparison.png`. Handoff: `tmp/drafts/handoff-spearman-face-fit-gpt6-astra.md`. Stop here for appearance review; MAA remains unchanged.
