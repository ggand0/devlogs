# 0096 — Levy archer face rebuild

2026-09-23, GPT-6 Astra. User liked the archer body but rejected the exposed face, especially from the front, and requested a reusable face scaffold following the levy reference.

Created `assets_dev/archer/textured_v2/` without modifying v1. Rebuilt the face with anatomical brow, nasal bridge, cheek, lip and chin rows. Generated an original skin albedo using built-in image_gen, preserved its exact PNG and prompt, and fitted its feature landmarks with shader texture coordinates. No reference pixels copied. New face module is separate from coif and clothing.

Result: 2,985 L0 triangles, including 250 face triangles; 1.80 m; 1,887 source and 3,655 exported vertices. Same material/atlas/part/pivot contract. Independent GLB inspection, image payload, color alpha, UV1 parts, triangle membership and scope checks pass. Verified unchanged 31 other geometry components, 18 source tiles and 59 old revision files. Examined exported front/three-quarter/profile and full-unit renders; generated matching before/after sheet and RTS scale checks.

Review files: `face_comparison.png`, `face_front.png`, `archer_review.png`, `validation.json`, README. Handoff: `work/handoffs/handoff-levy-archer-face-v2-gpt6-astra.md`. Gota responded positively to the revised face and subsequently reported that it apparently works. No transfer to other units, lower LODs, animation, engine changes, Cargo run, final promotion or commit was performed in this pass.

## Why this worked better than the earlier faces

Earlier iterations painted facial features with Python shapes, gradients and noise. That approach produced thin drawn-on eyes and eyebrows, a weak mouth and a continuous beard-shaped patch. Adjusting individual mesh proportions could not resolve those texture limitations. Continuing to tune that procedural painting was the wrong approach for the exposed face.

The main change was using built-in image_gen for an original facial albedo with natural eyelids, lips, skin variation and irregular stubble. The geometry was rebuilt at the same time to support the brow, nasal bridge, cheeks, lips and chin. Shader coordinates then fitted the generated texture's landmarks to the mesh; applying the image without that alignment would put features in the wrong places. A continuous face UV island retained approximately 335 × 440 pixels in the final 2048² atlas.

Matching front, three-quarter and profile renders of both exported GLBs made the differences visible without changing the camera or lighting. The complete unit increased from 2,941 to 2,985 triangles: only **44 additional triangles**. The improvement came from richer facial texture information, better geometry and their alignment, rather than a large polygon increase. These changes were made together, so their individual contributions were not measured separately.

## Workflow to reuse

1. Use the reference to identify concrete facial landmarks and proportions. Build the broad facial planes and silhouette in geometry.
2. Generate an original, evenly lit skin albedo with natural features and restrained stubble. Avoid copying reference pixels, strong directional shadows or highlights into the texture.
3. Fit the texture's actual eye, nose, mouth and chin positions to the mesh. Reserve enough atlas space for those features to survive baking.
4. Inspect the exported model straight-on as well as in three-quarter and profile views, then check full-unit and RTS screen sizes. Run the independent GLB and channel checks.
5. Preserve the generated raster and prompt alongside the build scripts. Rebuilding uses the saved image; requesting another generated image is not deterministic.

Reusable files are `face_geometry.py`, `generated_face.png`, `make_face.py`, the face projection branch in `materials.py`, and the face-island allocation in `build_archer.py`, all under `assets_dev/archer/textured_v2/`. The full generation prompt is in `face_generation_prompt.txt`. This is an exposed-face scaffold; destination units still supply their scalp, ears, neck and headwear. The texture retains some photographic feature shading, and this pass adds neither separate eyeballs nor a facial rig. In-game performance was not measured by this agent.
