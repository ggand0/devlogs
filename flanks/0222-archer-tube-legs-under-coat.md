# Archer tube legs under the coat

Written by GPT-6.

Gota preferred the original tube legs when covered by the coat and explicitly directed that they be used. lower_body_v3 restores their source positions and normals exactly and keeps their original knee/ankle weights. The pelvis attachment at the upper hose ring and the two swing-only coat joints remain from v2. This is the selected leg treatment, not an optional comparison variant. The earlier folders remain intact.

The wider source legs needed extra garment clearance. The coat receives up to 14 mm at its outer panels, fading inward and toward the waist, and 6 mm of front/back room fading to zero at the waist. Legs themselves are not narrowed or reshaped. The build reads skeleton_v1 and writes only lower_body_v3. No source or shipped model is changed.

The checker asserts bit-identical L0 hose positions and normals against skeleton_v1. It also verifies original 20 joints/pivots, unchanged upper body, UV/atlas/part IDs, triangle indices and L1/L2/L3 data. 22 joints; 2,987 / 680 / 250 / 56 triangles; ready width 0.787148 m. Shared clip validation and raw GLB inspection pass. Standing, ready and all 201 diagnostic times have zero detected cloth/leg edge crossings, including times interpolated between saved samples. The sweep is a sliding-foot test, not a gait, and does not prove clearance for future animations.

Review output is assets_dev/archer/lower_body_v3/review.html. The original model is shown only to show the coat change; its tube legs are also the legs used by the revised model. The player capacity commit from the previous turn is unchanged. Claude's spearman integration remains untouched. The new model remains local for L0 review; lower-detail coat work follows acceptance.

Inspected ready renders from front/oblique, side and back and the ordered motion sequence. The coat covers the original upper legs; no visible protrusion in these views. Surface boundary/winding counts equal v2 and the source. Maximum sole error through interpolated diagnostic frames is 0.00836 mm (the checker allows 0.01 mm); saved ready sole error remains below 0.000057 mm. This interpolation lift also applies to the unchanged diagnostic in v2; the older devlog's smaller sole figure described ready, not the full sweep.
