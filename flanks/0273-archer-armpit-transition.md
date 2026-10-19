# Archer armpit transition

Written by GPT-6.

The additional references are `refs/archer/male_shoulder.jpg` and `refs/archer/human_male.jpeg`. The first shows the anterior axillary fold continuing from the chest into the upper arm; the clothed figure shows a narrow, curved underarm recess with relaxed arms. The archer's neutral pose holds its arms farther from the torso, so matching the photographs' gap exactly would change the pose. V6 had a high, sharp notch and insufficient depth in front of the arm root.

## Geometry and deformation

Created `assets_dev/archer/arms_v7`; all preceding iterations remain in place. The front axillary boundary moves from (0.169, 1.350, 0.052) to (0.173, 1.338, 0.082) metres, mirrored across X. The underside moves from (0.173, 1.327, -0.003) to (0.178, 1.310, -0.005). The rear fold moves from (0.170, 1.352, -0.051) to (0.174, 1.340, -0.069). Front and rear intermediate vertices carry that depth into the chest and back.

The first sleeve contour lowers its inside point 35 mm, front-inner point 22 mm and rear-inner point 18 mm. The inside point also moves 12 mm outward to retain clearance from the chest. These are edits to the continuous chest/sleeve surface, without extra connecting solids. The shoulder crest, outer sleeve width, forearms and hands retain v6's shape. Triangle counts do not change.

The first shape export crossed the right chest during the early draw. The first contour previously followed the upper arm almost completely, including the cloth directly beside the chest. Its inner point now has 60% upper-arm influence and the adjacent folds 80%; the remainder follows the chest. The outside continues following the arm. The final exported GLB passes the sampled motion checks.

## Validation

L0/L1/L2/L3: 2,994 / 680 / 250 / 56 triangles. Height 1.80 m, 22 joints. The replaced surface contains 820 triangles in one closed component, with zero open edges, nonmanifold edges, inconsistent shared-edge winding or degenerate triangles. Neutral and 174 sampled stand, ready, draw, raised-aim and release poses show zero detected crossings or seam openings in that surface. The check samples edge/triangle intersections; it does not prove continuous collision freedom or containment.

Joint nodes and pivots are unchanged: shoulders (±0.205, 1.435, 0), elbows (±0.290, 1.155, -0.010), wrists (±0.350, 0.925, 0.130), in metres. Part coverage is in `validation.json`. Retained L0 attributes, retained atlas pixels and L1-L3 are unchanged. Both raw GLB and surface inspectors ran. Source defects outside the replacement surface remain outside this change.

Model SHA256: `2025f2d9368921a5d2e51fc5436698c92c6d13d5eb80d91c7ee487c1d3ba916a`.

## Review scope

The review starts with matching close front/rear armpit comparisons, followed by full front/reference comparison and textured motion. Four cardinal views use the same scale. The photographs remain in the ignored `local_reference/` folder and are not embedded in tracked comparison images. The original coif remains; shoulder cloth is absent while the arm shape is reviewed. Lower-detail meshes still use the original source arms. No public code, installed model or shot JSON changed. No iteration is committed or shipped.

Final media checks: the fixed-view video has 87 frames at 24 fps, 1024 by 1096. Neutral, level-shot and raised-shot orbits have 192, 288 and 432 frames at 24 fps. All four videos decode without errors. All 3 orbit manifests match the final GLB hash, and all 38 local review links resolve. Inspected full-size front/rear/both-side clay and textured renders, draw poses, all twelve consecutive-frame sheets and all three orbit contact sheets. No real-time playback or in-game review is claimed. V4, v5, v6 and the installed archer retain their preceding hashes.
