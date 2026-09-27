# 0151 Mature oak colour and bark revision
Written by GPT-6 Astra.

The initial request asked for leaves with stronger green contrast against the grass and reported white trunk lines in debug2–debug4. I interpreted “brighter” too literally and generated lighter green source atlases. Gota clarified the intended direction as darker Valheim-like green, then limited this review to the mature oak. The runtime therefore uses tree_set_v2 only for mature_oak; upright oak, leaning oak and silver birch retain oak_v2/tree_set_v1. Earlier v2 builds of those trees remain unused on disk.

The mature oak uses the generated green oak atlas with linear vertex tint (0.36, 0.52, 0.60), multiplied by the existing per-spray variation. Blender preview shading and far-colour baking now include vertex colour, matching the game. The reference WebP was converted to resources/vegetation/velheim_meadow_trees.jpg without replacing the WebP.

Bark previously shared a MASK material and a 1254² atlas with foliage. Inspection showed coarse bark mip alpha contaminated by transparent adjacent regions (e.g. values below 128 at small levels). Both sides rendered, allowing lit interior surfaces to show through cutout holes. The new asset separates opaque bark into its own texture/material and keeps foliage cutout. Bark UVs mirror continuously around the tube and alternate at segment rings to remove colour discontinuities. Bark pixels are packed from the original source atlas; no recolouring. Runtime supports either the legacy combined primitive or separate foliage/bark primitives, with two child draw entities for the latter. Far cards use a single cutout draw and hide the spare bark child. Opaque mip generation skips alpha-coverage scaling.

Raw GLB checks pass. Mature oak: 2912/636/4 triangles, L0 height 13.719443 m, L1 height 13.836053 m, identity transforms, UV0/UV1 and COLOR_0 VEC4 retained. Every exported vertex position matches v1 exactly. Material splitting can change the exporter’s triangulation diagonals, so the verifier compares vertex positions rather than claiming byte-identical triangle ordering. Opaque bark embedded alpha is 255 throughout. The standalone glb_inspect route is also run on the final GLB.

Validation: opt-dev build, strict all-target clippy, 3 mip tests pass (transparent RGB, odd atlas edges, opaque bark). Source diff is vegetation rendering only; no sim inputs, terrain heights, random sources or FL switches change. Near/middle draw count rises from one to two per revised tree. No forest performance claim.

Review: tmp/shots/0017_vegetation-colour-seams/index.html, matched before/after close and lineup views, 250/900 m stills and 22-second zoom/orbit. The branch still has no cast tree shadows. Review is pending; stop here before extending the colour treatment to the other trees.

The 22-second motion recording was inspected as a 2 Hz sequence (44 samples). Logs confirm mature oak L0 → L1 → card → L1 → L0, with the bark child returning on the mesh levels. Matched battle/far captures are included. Static review links were checked locally.
