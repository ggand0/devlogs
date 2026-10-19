# Archer arm front, rear and side corrections

Written by GPT-6.

2026-10-18. The user rejected arms_v2 for excessive thickness in several views and unnatural arm shapes, then explicitly required inspection of front, rear and side views before another review. The preceding increase from 112 to 160 mm in upper-sleeve width also increased depth from 132 to 164 mm; that increased volume without establishing a convincing cross-section. The side render showed a swollen sleeve, and the elbow folds formed thick ledges. Those defects should have blocked the preceding handover.

## Changes and actual inspection

New iteration `assets_dev/archer/arms_v3/`. All rejected folders remain on disk unchanged. Reduced sleeve width and depth independently, redistributed the sleeve stations, reduced the flared hem and made the linen folds much shallower. The connected wrist and hand construction remains. The lower torso and armhole from v2 are retained.

Inspected individual full-size front, rear, right-side and left-side clay renders, then both clay and textured four-view sheets. Added rear fill lighting to make the back surfaces legible. The page opens with those four views rather than an oblique comparison. It also includes matching views of the rejected version and front, side and rear reference crops. The source reference has different posing and perspective; no pixel match is claimed.

The side views exposed the protruding thumb shape. A first fold correction made it appear as a thin edge from the front, so that internal draft was not handed over. Rebuilt the thumb from its palm root with oriented sections following the bend and added a short tip bevel. An initial section orientation folded through itself; the knuckle section now follows the bend bisector. Re-rendered and inspected all four views after these corrections. The hands are still low-poly gripping forms; individual finger detail remains unfinished.

Clothed cross-sections, v2 to v3, width by depth in millimetres:

| Section | v2 | v3 |
|---|---|---|
| Upper sleeve | 160 × 164 | 129 × 123 |
| Sleeve hem | 156 × 151 | 119 × 109 |
| Proximal forearm | 123 × 107 | 105 × 87 |
| Wrist | 58 × 45 | 53 × 41 |

These compare corresponding features; some axial stations moved. They are mesh measurements, not measurements inferred from the illustration.

## Validation and scope

L0/L1/L2/L3 contain 2990/680/250/56 triangles. Height is 1.799999952 m. The generated upper body, both arms and hands form one closed 816-triangle component. Zero open, nonmanifold or inconsistent-winding edges, zero degenerate triangles, and zero detected edge/surface self-crossings in the neutral pose. The final GLB passed the iteration checks and glb_inspect.py; inspect_surfaces.py also produced its per-part report. Shared part boundaries are continuous geometry, so their isolated part boundaries are not exposed mesh holes.

The original 22 joint nodes, pivots, skin, retained L0 attributes, retained texture texels and L1-L3 geometry are unchanged. Shoulders remain at (+/-0.205, 1.435, 0), elbows at (+/-0.290, 1.155, -0.010), wrists at (+/-0.350, 0.925, 0.130) metres. Full part coverage and all joint origins are in validation.json and build_report.json.

Final SHA-256: b8e8ee7fe9ff7c57600e20a72b5b47088bf2c8168bfa5d5f845d3eca4313e00d. The validation and build reports match the file hash. Source SHA-256: d2ec53996146b9c7e0649b74a8a6a97dfd6e6aa570d04702c1bb6f188ec823c6.

This remains the first, neutral-shape checkpoint. The broad provisional wrist weights remain to be replaced in the deformation stage. No animation or in-game improvement is claimed from neutral renders. The bracer and finger detail, lower-LOD updates, smaller hood and full shoulder cape remain for later steps. The bow is hidden only in the shape renders, not removed from the GLB.

No public asset or engine code changed, and no commit was made. Updated the iterations table with the v2 rejection and v3 review. Do not ship this as an animation-ready asset.
