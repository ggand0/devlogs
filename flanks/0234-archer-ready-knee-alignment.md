# Archer ready knee alignment

Written by GPT-6 Astra.

Gota reviewed standing positively and marked the ready leg gap and knee directions in refs/archer/ready0.png. Inspected that image and matching M2TW views. Built standing_ready_v2 with ankle spacing reduced from 0.32 to 0.29 m across and 0.62 to 0.52 m front to back. Raised the pelvis 0.02 m and brought knee bend directions inward. Both IK targets are reachable. Standing, the model and upper-body rotations are unchanged.

Ready width is 0.787 m. All four levels have flat grounded soles; L0 coat/leg crossings are zero. Existing tube attachment rings overlap inside the tunic near the pelvis (14 edge-surface intersections before, 16 after); none occur on the lower legs. Geometry and shared clip validation pass. See geometry_validation.json for bounds and numerical limits.

The review page includes before/after views, both 24-second orbits, seven matching M2TW views and fresh measurements. Three ready reference differences are retained and explained: upright bow 46.2 degrees, right foot facing 18.9 degrees, forward foot spacing 14.4 percentage points of leg length. The new stance is independently authored for the reviewed silhouette. No engine files changed. Visual review of v2 is pending.

Both 24-second videos decode and play through a complete browser loop. Main and reference pages pass link, image and source-hash checks. Viewed the before/after sheet, matching reference views and full orbit sequence.

## Review and next item

Gota says standing_ready_v2 is good enough for now. Keep both standing and ready as the current endpoints. The quiver position should later move toward the M2TW archer placement so the arrow retrieval makes sense; that model adjustment is deferred. Next item is the 1.5-second draw animation, ending at full draw, on the current model.
