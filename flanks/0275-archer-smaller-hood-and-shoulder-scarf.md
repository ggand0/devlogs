# Archer smaller hood and shoulder scarf

Written by GPT-6.

2026-10-18. Restored the red wool hood and shoulder-covering scarf over the provisionally approved arms_v7 foundation, in `assets_dev/archer/hood_v3/`. The consolidated arms history is devlog 0274. V7 was committed in the private assets repository on main as `1f2b78c`, “Add the archer arms v7.” Hood_v3 is a new review iteration, not a replacement of the public asset.

## Reference and proportions

The original `refs/archer/levy_archer_reference.png` has a red raised-hood variant. Its shoulder cloth descends over the shoulder caps; the main front/rear/side views also show a shoulder-covering wool cape. Hood_v1 had outward shoulder wings, while hood_v2 shortened the cloth enough to lose that coverage. This pass retains v7's torso and arms and restores the coverage around them.

The hood width changes from 252 to 237 mm, a 6.0% reduction. Its depth changes from 292 to 272 mm, a 6.8% reduction. The crown stays at 1.80 m; this is a width/depth reduction, not a reduction of the character's height. The face opening follows the retained head. A rounded crown ring prevents the reduced hood from ending in a pointed cap.

The scarf spans 654 mm over the two shoulder caps, compared with the 435 mm scarf in hood_v2. Its hem sits at about 1.300 m at the front, 1.305 m at the back and 1.345 m at the sides. The cloth slopes from the neck over the shoulders. It remains a simple low-poly drape; small folds and surface detail are much simpler than the reference.

## Construction and internal corrections

Removed only the coif and tie components from part 18, 362 triangles, keeping the 250-triangle face/head. Added a 172-triangle closed hood and a 192-triangle closed scarf. Every retained v7 vertex attribute is unchanged. The hood uses the head joint, and the scarf uses the existing chest/upper-arm parent-child pairs. No joints or animation clips were changed.

- A joined hood/scarf neck distorted when the head and chest counter-rotated. Separate closed shells with covered collar overlap allow the head to turn independently. Their overlap is intentional and is not included in a claim of intersection-free contact between the two garments.
- The first scarf drape crossed the tunic and moving upper sleeves. The front shoulder geometry was moved outward slightly, the crest was given 55% arm influence, the rear crest 18%, the front inner drape 30% and the rear hem 55%. Centre-line cloth stays on the chest; a near-zero floating-point X value had incorrectly assigned a centre-back point to the left arm, and the side selection now requires more than 25 mm from the centre.
- The front fold changed too sharply from the arm to the chest. Its lower shoulder row now has 88% arm influence and the upper inner point 22%. A weight sweep checked both self-intersections and body clearance, rather than treating a clean exterior alone as sufficient.
- A simplified inner scarf surface crossed the outer cloth during drawing. The lining now has matching topology and skin weights. Its approximately 4 mm offsets are fitted against the adjacent outer-face normals through 245 neutral and bow samples. The saved `lining_offsets.json` is a build input, not a render-only correction.
- Redundant side/back hood rows were merged to keep the complete character below 3,000 triangles while restoring the full scarf lining. A concave opening polygon initially produced three self-crossings; changing its triangulation corrected them.
- The full equipment check found the lower bowstring passing through the front hem near full draw. The front quarter hem points were raised 33 mm, keeping both shoulder tips covered. The bow-side front hem now uses 15% arm influence, and the draw-side uses 43%; this lets the front drape hang closer to the chest while the sleeve lifts beneath it. The final 256-sample check includes both garments against the bow and string and detects no crossings.
- The rear wool texture had a visible centre seam. Mirrored angular UV mapping makes the sampled texture continuous across the back. Retained atlas pixels are unchanged; only an unreferenced rectangle is used for new wool and hood lining.

All changes are in the new hood_v3 folder. Earlier rejected iterations and v7 remain on disk unchanged.

## Export and validation

Saved model SHA-256: `1a4ddbd184d564dc45594b5166cc9027fb48b0d000744869f8d1ddbb916ef8a3`.

| Item | Result |
|---|---|
| L0 / L1 / L2 / L3 triangles | 2,996 / 680 / 250 / 56 |
| Height | 1.800 m |
| Original joints | 22 |
| Retained source vertices | 2,940, all attributes unchanged |
| New hood / scarf triangles | 172 / 192 |
| New part ids | Hood 18, scarf 19; all new triangles carry their assigned part |
| New shell open / nonmanifold / wrong-winding / degenerate counts | 0 / 0 / 0 / 0 for both shells |
| New shell signed volumes | hood 0.001736 m³; scarf 0.001167 m³ |
| Motion samples | 256 |
| New garment self-crossings | 0 detected in those samples |
| Garment exterior versus head, torso and arms; both garments versus bow/string | 0 detected in those samples |

The sample set includes neutral, 81 head turns through ±55 degrees, stand, ready, draw, raised aim, and both releases. The minimum new-triangle area ratio is 0.0775 during drawing. Checks use the game's joint-centred two-joint deformation, not Blender linear blend skinning.

Head pivot (0, 1.485, 0), chest pivot (0, 1.035, 0), shoulders (±0.205, 1.435, 0), elbows (±0.290, 1.155, -0.010) and wrists (±0.350, 0.925, 0.130), in metres. Nodes, skin definitions, pivots and L1–L3 arrays match arms_v7 exactly. The original 22-joint rig is retained despite the older spec's 20-joint target.

The raw GLB inspector and surface inspector were run on the saved file; their reports are in the iteration. Whole-model part partitions retain source openings and other source topology defects; the zero-error topology result above applies to the two newly built closed shells. The concealed scarf lining still intersects the covered torso in some drawing/aim/release samples, up to 24 detected edge crossings. The exterior is checked separately and stays clear. Sampled intersection tests do not prove continuous collision clearance or containment. The pre-existing early-draw string/arm contact is not fixed by this garment pass.

## Review scope

The review page includes a same-scale front/side hood comparison, the red variant from the original reference, four textured cardinal views, clay views, fixed-camera textured motion and textured neutral/level-shot/raised-shot orbits. The older hood comparison uses hood_v2, whose underlying body differs, so it is a hood comparison rather than a body comparison.

This checkpoint concerns L0 garments. L1–L3 still use the preceding shape and need later matching. No public model, build script or animation JSON was replaced. Hood_v3 stays uncommitted until review. Textured image sequences and full-size views are inspected for this checkpoint; no real-time playback or in-game review is claimed.

Final preview checks: four videos decode without errors. Fixed views contain 87 frames at 24 fps; the neutral, level-shot and raised-shot orbits contain 192, 288 and 432 frames at 24 fps. All manifests identify the final model hash. All 31 local review links resolve. Inspected the four cardinal views, drawing poses, consecutive motion frames and all three orbit contact sheets, with full-size orbit frames for the scarf/string contact.
