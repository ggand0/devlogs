# Archer shoulder cloth fit
Written by GPT-6.

The hood itself can stay; the coif was also a valid direction. The marked screenshot in refs/archer/hood_review0.png points to shoulder-cloth peaks and excess width. This revision remains at checkpoint 1. Sleeve work waits for the next review.

## Geometry

The separate iteration is assets_dev/archer/hood_v2. The preceding hood_v1 folder is preserved, with 37 direct-file checksums verified unchanged. The visible hood triangles match hood_v1. Its concealed inner crown uses 32 fewer triangles; the mantle uses those triangles for two more exterior rings. Hood 316 triangles, mantle 224, total L0 2,982. L1/L2/L3 remain 680/250/56.

The mantle width changes from 0.638 m to 0.435 m. It ends above the upper sleeves instead of flaring beyond the shoulder outline. Its side crest is capped at 1.543 m. The fitting script constrains projected triangle overlaps against the original torso and 36 sampled draw, elevated aim and release poses, with 9 mm exterior clearance. A broader cape could clear the torso but the shooting sleeves pierced it; the shorter hem resolves that without altering sleeves or animations. The mantle follows the chest joint.

## Verification

GLB inspector confirms the required stored attributes. The skeleton, 22 joints, pivots, retained L0 attributes and all distance meshes match the source. Height remains 1.80 m. Atlas edits remain inside the unused reserved rectangle. Both new shells have positive volume, zero open edges, nonmanifold edges, winding conflicts or degenerate triangles.

All tested surface pairs have zero crossings over 255 sampled poses: head turns, stand, ready, draw, aim up, release and release up. This includes cape exterior against torso and upper arms, the entire cape against forearms, and concealed lining against torso. These are sampled edge-triangle checks, not continuous clearance or containment proofs.

The review contains full-figure before/after views, four close views, posed stills and a ten-second side-by-side motion comparison. The existing sleeves and equipment remain. This is an L0 shape review, not a shipped asset. No public game asset or source code is changed. No iteration is committed while the review is pending.

The shared-GPU guard delayed Blender several times while games were running. Each build and render started only after an elevated guard call reported no game running.

Review validation: all local links resolve, all 240 video frames decode, and headless Firefox plays through a full loop. Browser playback captures were inspected across head turns, level draw and elevated aim. The full-figure comparison is silhouette.png; the review page is review.html.
