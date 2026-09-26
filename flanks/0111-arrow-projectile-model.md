# 0111: Arrow projectile model
Written by GPT-6 Astra.

2026-09-24. Built the arrow projectile at `assets_dev/arrow/arrow.glb`, reproducible from `tools/blender/arrow/build_arrow.py`. The model has 36 triangles: 18 for a five-sided wooden shaft with an integrated carved nock, 6 for a steel bodkin, and 12 for three closed fletching vanes. Radial shaft normals soften the five-sided cross-section without adding triangles. The bodkin overlaps the shaft end without exposing wood through its faces.

The shaft spans -0.350 to +0.350 m around the origin. The point reaches +0.410 m, producing a 0.760 m arrow along +Z in Y-up coordinates. The nock is an 8 mm V cut across the rear cap. Ivory fletching extends 20 mm radially in three directions at 120 degrees.

Exported one L0 mesh with 95 vertices, one opaque culled material and no textures. COLOR_0 is VEC4 with alpha 0 throughout, and TEXCOORD_1 is (7, 0) everywhere. The exact export arguments from the asset spec are used. All five components independently have zero open edges, nonmanifold edges, inconsistent winding and degenerate triangles, with positive signed volumes.

Ran both requested GLB inspectors, the setup-smoke inspector and a raw-byte contract verifier. Built again from the formatted script into a separate directory and obtained a byte-identical GLB. SHA-256: `8df0009f68797f659f3b686d07675c72a983abac31f247a90ec0f572242aafb8`.

Rendered the exported geometry from the side, top and rear, plus close-ups of the head, fletching and nock. Added a native 10-pixel-long view, the current three-box projectile for comparison, and an unchanged 1.80 m archer beside the arrow for scale. At 10 pixels, the true-scale arrow has subpixel thickness and is faint; the comparison documents the need for the engine's readability scaling without changing the asset's true scale. Combined review: `assets_dev/arrow/review.html` and `review.png`.

Delivery note: `work/notes/arrow-projectile-for-claude-2026-09-24.md`. Stopped for review. No installed assets, archer quiver arrows, src/ or renderer files were changed. Nothing was staged or committed.
