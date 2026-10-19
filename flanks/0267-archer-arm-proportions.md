# Archer arm proportions

Written by GPT-6.

2026-10-18. First checkpoint of the proposed correction after arms_v1 was rejected. The user called out thin arms and bad deformation in archer_review0.png and archer_review1.png. The agreed sequence is neutral proportions first, then elbow/wrist deformation, then bracer and hand finish followed by a smaller hood and full shoulder cape. Stop for review after the first two checkpoints.

## Changes

New iteration `assets_dev/archer/arms_v2/`; rejected versions remain untouched. The 22-joint release_aim_v2 model is the retained asset source. Replaced upper sleeves and forearms with directional contours distinguishing the inner seam, outer arm, front mass and rear mass. Added substantial cloth allowance, an angled hem and gathered linen around the elbow. The upper-arm inner contour is not compressed to avoid intersections. The armhole is raised and its lower torso attachment shaped inward to join the fuller arm without overlap. Hand geometry shares the wrist boundary, replacing capped overlapping pieces.

The first neutral draft had twelve detected edge/surface crossings at the underarm. Localized them to the torso side and corrected that connection without reducing arm thickness. The final neutral mesh has zero detected crossings. Hands remain simplified gripping forms; the bracer remains a material/shape placeholder, without the construction detail planned after deformation review.

Measured contour widths, previous to new: upper sleeve 112 to 160 mm; sleeve hem 107 to 156 mm; proximal forearm 94 to 123 mm; wrist 46 to 58 mm. These are local clothed cross-sections, not inferred anatomical measurements from the image. The upper sleeve and hem stations were relocated, so the comparison describes the corresponding features rather than the same axial coordinate. The original reference has different perspective and posing; it is shown for clothing and proportion judgment, not pixel matching.

## Export and checks

L0/L1/L2/L3: 2974/680/250/56 triangles. Height 1.799999952 m. The new connected torso, both arms and both hands use 800 triangles in one closed component, with zero open edges, nonmanifold edges, inconsistent edge winding or degenerate triangles. Neutral edge/triangle crossing checks return zero. Exported GLB bytes were checked with glb_inspect.py and the iteration's check_shapes.py. Original nodes, skin, pivots, retained L0 attributes and retained atlas texels are unchanged; L1-L3 retain the previous geometry. Part channels, VEC4 color and normalized parent-child weights are present.

Per-part surface inspection also ran. Part boundaries across the connected upper body are shared geometric seams, so isolated part reports have boundary edges; the combined new surface is closed. No new joints or pivots were introduced.

Joint origins in metres, unchanged: shoulders (+/-0.205, 1.435, 0), elbows (+/-0.290, 1.155, -0.010), wrists (+/-0.350, 0.925, 0.130). Full part coverage and all joint origins are in validation.json and build_report.json.

Output SHA-256: e8e6e39696f419d0b56386aeac75a7a7199cb8055c896b2b6cf52cb53071c83f. Source SHA-256: d2ec53996146b9c7e0649b74a8a6a97dfd6e6aa570d04702c1bb6f188ec823c6.

## Review and limits

Rendered the same identity pose and camera scale for arms_v1 and arms_v2, in clay and cloth from front, side, back and oblique views. Inspected the generated images and comparison sheets. The local page has fifteen valid local links. The reference comparison is in ignored local_reference/; our neutral comparison is neutral_comparison.png. The bow and string are hidden in the review renders to expose the hands and forearms, but remain in the GLB.

This is only the first checkpoint. Skin weights are provisional, including the previous broad wrist weighting, and no animation validation or in-game test is claimed. The next step is to inspect joint axes and improve bend weighting using isolated elbow/wrist motion and the original draw/release clips. The lower LOD silhouettes, bracer construction, detailed finger shapes, hood and full shoulder cape remain for their later steps. Do not ship this candidate before those checks and reviews.

No public assets or engine code changed; no commits. Updated the iteration table with the rejection of arms_v1 and pending shape review for arms_v2.
