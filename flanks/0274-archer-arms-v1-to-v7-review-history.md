# Archer arms v1 to v7: issues, corrections and review outcomes

Written by GPT-6.

2026-10-18. This consolidates the complete arm rework history from the user reviews, iteration scripts, exported validation records and devlogs 0266–0273. V7 received provisional shape approval: “I suppose it looks good enough for now.” The next requested stage is the shoulder-covering scarf and a slightly smaller red hood. This is approval of the exposed-arm foundation, not the complete character or its distant meshes.

## Why the arms were rebuilt first

The source archer used separate spherical joint covers and tubular arm sections. The original reference calls for a continuous human silhouette under loose wool sleeves, gathered linen, tapered forearms, narrow wrists and a fitted left bracer. Hood v1 had raised shoulder wings. Hood v2 reduced the cape's shoulder coverage to avoid the moving arms, which contradicted the reference and the explicit requirement to cover the shoulders. The user required the entire arms to be fixed before cloth hid the anatomy, and a smaller hood afterward. Hood v1/v2 and every rejected arm folder remain on disk.

## arms_v1: connected geometry, but insufficient volume

Problem addressed: separate ball joints and tubes, mechanical shoulder/arm junctions and an incorrect order of work that covered defects with shoulder cloth.

Changes: removed 813 source triangles comprising the upper tunic, arms, shoulder balls and cape. Rebuilt the upper tunic and sleeves as one 640-triangle connected surface, stitched to the exact original waist boundary. Added directional sections for shoulder, upper arm, elbow, forearm and wrist; wool ends above the linen. The left bracer follows the forearm. Each hand was a separate closed 92-triangle grip with a connected thumb web. The original coif remained while the cape was removed for inspection.

Deformation issues found while building: underarm loops folded, elbows compressed, wrists twisted and thumbs intersected themselves. Corrected section orientation, local geometry and parent/child weights. The final new surfaces passed 174 sampled poses without self-crossing. New arms versus head and sleeves versus bow limbs had no sampled crossings. A pre-existing string/arm crossing remained during early draw, 0.125–0.200 seconds: source 24–26 crossings, candidate 12–14. The shot JSON was not changed, and those counts were not a clean equipment-clearance result.

Outcome: rejected after archer_review0.png and archer_review1.png. The user acknowledged the removal of sphere joints but said the arms were too skinny and that thin volume exposed poor joint deformation. The model still did not resemble the reference. Closed geometry and passing intersection checks did not establish convincing anatomy or cloth proportions.

Measurements: 2,998 / 680 / 250 / 56 triangles. New surfaces total 824 triangles in three closed components, 640 + 92 + 92. Height 1.80 m. Twenty-two original joints and pivots retained. See devlog 0266 and arms_v1 validation records.

## arms_v2: overcorrection to excessive thickness

Problem addressed: v1's thin sleeves and forearms, weak clothed volume and disconnected wrist/hand construction.

Changes: directional front/back/inner/outer arm contours, substantial cloth allowance, an angled wool hem, gathered linen at the elbow, and a raised armhole with a shaped lower connection into the torso. Hands now share the wrist boundary instead of meeting through capped overlaps. Twelve neutral underarm crossings in the first draft were traced to the torso side and corrected locally without simply thinning the arm.

Clothed widths increased from v1 to v2: upper sleeve 112 to 160 mm, hem 107 to 156 mm, proximal forearm 94 to 123 mm, wrist 46 to 58 mm. Sleeve depth also grew to 164 mm. These are mesh dimensions at corresponding features, not measurements derived from the reference photograph.

Outcome: rejected. The user said some views were much too thick and the figure looked unnatural, and explicitly demanded front/rear/side inspection instead of judging the diagonal view. Volume had been increased in width and depth together without resolving the section shape. The bulky elbow folds also created ledges. These defects should have prevented presentation.

Scope and checks: this was a neutral-shape checkpoint, not a validated deformation revision. Skin weights remained provisional, including the broad wrist blend. No animation improvement was established. The bracer was still a placeholder. 2,974 / 680 / 250 / 56 triangles; the new 800-triangle torso/arms/hands surface was one closed component, with zero detected neutral crossings. See devlog 0267.

## arms_v3: better section sizes, wrong shoulder structure

Problem addressed: v2's swollen sleeves, excessive side-view depth, flared hem and thick linen elbow ledges.

Changes: reduced width and depth independently, redistributed sleeve stations, reduced hem flare and made linen folds shallower. Added rear lighting and made front, rear and both side views the primary review. Side inspection exposed a protruding thumb. An internal correction made the thumb look like a thin edge from the front; it was rebuilt from its palm root with oriented sections and a short tip bevel. A self-crossing knuckle section was corrected to follow the bend bisector.

Clothed sections, width × depth in millimetres:

| Feature | v2 | v3 |
|---|---|---|
| Upper sleeve | 160 × 164 | 129 × 123 |
| Sleeve hem | 156 × 151 | 119 × 109 |
| Proximal forearm | 123 × 107 | 105 × 87 |
| Wrist | 58 × 45 | 53 × 41 |

Outcome: rejected after archer_review2.png. The user identified the unnatural horizontal piece between torso and arm. The inherited topology still projected sideways from the chest before turning down into the sleeve. Resizing the sections could not repair that structure. The instruction was to create a human figure, not connecting pieces, and to fix other visible defects as well.

Scope and checks: still a neutral checkpoint with provisional wrist weights. 2,990 / 680 / 250 / 56 triangles; one closed 816-triangle new surface; no detected neutral crossings. Neutral renders did not prove animated deformation. See devlog 0268.

## arms_v4: shoulder reconstruction and deformation repair

Problem addressed: the horizontal torso-to-arm connectors, joint compression/torsion and unfinished forearm/bracer construction.

Changes: removed the lateral connector rows and rebuilt the armhole around the shoulder cap and front/rear axillary folds. The collar surface slopes over the shoulder pivot; the first sleeve section descends along the upper arm. Added collar-to-shoulder support and a curved outer shoulder edge. A pointed rear overhang appeared in the nocking pose despite clean intersection checks; its support geometry was corrected and inspected again.

The sleeve uses different front/back/inner/outer profiles and independent width/depth. Upper sleeve 129 × 127 mm, hem 119 × 111 mm, proximal forearm 107 × 91 mm, wrist 53 × 41 mm. Extended the leather bracer toward the wrist and added bound edges and stitches in its atlas tile. Replaced mismatched torso/sleeve UV scales with consistent cloth projection.

Elbow weighting follows the bend bisector across 180 mm. Wrist rotation blends across 100 mm through the hand base and thumb. Shoulder rotation spreads into the upper chest and back with separate front/top/rear support weights. Pivots and clips remain unchanged; the correction was geometry and weighting, not a joint remap.

Review sequence: the user said it might be alright but requested textured animated orbits rather than clay alone. Added neutral, level-shot and raised-shot orbits, plus fixed front/rear/both-side motion. After seeing animation, the user said it looked “fine ish” in motion but the front balance and shoulders still looked wrong. The new tunic-only reference and archer_review3.png exposed a high, broad shoulder shelf above a relatively narrow chest. The linen also read as a long smooth taper. This was further work, not approval to ship.

Checks: 2,994 / 680 / 250 / 56 triangles; one closed 820-triangle surface; zero detected neutral crossings, zero crossings and seam openings in 174 sampled poses. Minimum sampled triangle-area ratio about 0.116. Textured frame sequences and orbit contact sheets were inspected; no real-time playback or in-game review was claimed. See devlogs 0269 and 0270.

## arms_v5: fuller chest, but rounded shoulders and a bell-shaped sleeve

Problem addressed: v4's high/wide shoulder balance, narrow upper chest and long smooth linen taper, using levy_archer_reference_tunic.png and archer_review3.png.

Changes: shoulder crest lowered from 1.470 to 1.455 m and moved inward from ±0.205 to ±0.177 m. The actual vertical change was 15 mm, correcting a progress message that said 25 mm. Outside shoulder boundary lowered from 1.428 to 1.411 m and moved from ±0.254 to ±0.237 m. Upper chest half-width increased from 0.158 to 0.180 m and depth from 0.122 to 0.130 m. Sleeve centres moved inward and the wool flared toward its hem. Added a right linen cuff fold; mapped linen length by height rather than row number to reduce texture stretching. The left forearm stayed fitted to its bracer.

Internal problems: the first geometry crossed at the underarm in neutral and early draw. Moving sleeve rows and narrowing the first inner contour removed the intersections, but also helped create a pinched sleeve root. A rear shoulder fold during draw was corrected by reducing rear crest arm influence from 0.25 to 0.15.

Outcome: rejected in archer_review4.png. The user marked the shoulder as too round and the sleeve root/flare as bad, and explicitly marked v4's sleeve as good. The reference has a shallow shoulder line followed by a definite downward turn. Treating slope as a rounded dome and narrowing the root while flaring the hem moved the shape away from that reference. Passing collision checks did not make these changes visually correct.

Checks: 2,994 / 680 / 250 / 56 triangles; closed 820-triangle replacement; 174 sampled poses without detected crossings or seam openings. Same joints, pivots, retained data and distant meshes. See devlog 0271.

## arms_v6: shoulder plane and sleeve restored, armpits still pinched

Problem addressed: v5's rounded shoulder dome and narrow-root, flared-hem sleeve.

Changes: a shallow shoulder plane with a visible downward turn. Crest at (±0.218, 1.451, 0) m, joining the neck radius 0.060 m at height 1.500 m and the outer sleeve root at (±0.254, 1.415, -0.002) m. Separate front and rear support points shape the chest and scapular transition. Restored v4 upper sleeve stations at shoulder-to-elbow fractions 0.29, 0.52, 0.73 and 0.83, with radial widths 0.064, 0.066, 0.061 and 0.060 m. Removed v5's inward centre shift and narrowed inner first-row contour. Retained the fuller v5 chest and linen improvements.

An early export folded behind the drawing shoulder at 1.05–1.125 seconds. Reducing the rear support's upper-arm influence from 0.25 to 0.18 resolved the detected fold without changing the neutral outline.

Outcome: “Better,” but the user still found the armpits wrong and supplied male_shoulder.jpg, then human_male.jpeg. The remaining problem was a high, sharp underarm notch and a hollow in front of the arm root. The chest-to-upper-arm fold lacked enough depth, while the sleeve's first inner contour sat too high. This was a local correction request, not a reason to round or broaden the shoulders again.

Checks: 2,994 / 680 / 250 / 56 triangles; closed 820-triangle replacement; 174 sampled poses with no detected crossings or seam openings. Inspected four cardinal views, posed views, twelve motion sequence sheets and three textured orbit contact sheets. See devlog 0272.

## arms_v7: chest-to-arm folds and underarm deformation

Problem addressed: v6's high notch, recessed front arm root and incomplete chest/back transition, using the bare torso and fitted-shirt photographs. The photographs have arms closer to the body than the archer's neutral pose, so their gap is not copied literally.

Changes, mirrored across X: front axillary boundary (0.169, 1.350, 0.052) to (0.173, 1.338, 0.082) m; underside (0.173, 1.327, -0.003) to (0.178, 1.310, -0.005); rear fold (0.170, 1.352, -0.051) to (0.174, 1.340, -0.069). Adjacent vertices carry the depth into the chest and back. First inner sleeve point lowered 35 mm and moved outward 12 mm for torso clearance; its front/rear neighbours lowered 22/18 mm. Shoulder outline, outer sleeve width, forearms and hands retain v6's shape.

The first shape export intersected the right chest during early draw because the inner sleeve still followed the upper arm almost completely. Its inner point now uses 60% upper-arm influence, adjacent folds 80%, with the remainder on the chest. The outside continues following the arm. Final exported checks found no crossings in the sampled motion.

Outcome: provisionally approved as good enough for now. This is the foundation for restoring the scarf and smaller hood. No further unsolicited arm reshaping is planned in the hood stage.

Checks: 2,994 / 680 / 250 / 56 triangles, 1.80 m, 22 original joints. One closed 820-triangle replacement; zero neutral crossings or topology errors and zero detected crossings or opened seams across 174 poses. Four videos decode: fixed views 87 frames, neutral orbit 192, level orbit 288, raised orbit 432, all 24 fps. All 38 review links resolve. Four cardinal views, posed stills, all twelve consecutive-frame sheets and three orbit contact sheets inspected. See devlog 0273 and arms_v7/review.html.

## What every version preserves, and what remains

All seven versions retain the canonical release_aim_v2 rig and clips. Shoulder pivots are (±0.205, 1.435, 0), elbows (±0.290, 1.155, -0.010), wrists (±0.350, 0.925, 0.130), head (0, 1.485, 0), in metres. Retained vertex attributes and atlas pixels are checked against the source. L1/L2/L3 still have the original geometry at 680/250/56 triangles; their silhouettes have not yet been rebuilt to match the new arms. No public game asset or animation JSON was replaced during the arm reviews.

The repeated failure was treating topology or intersection checks as evidence of a convincing human/clothing shape. V1 removed spheres without enough mass. V2 increased mass in every direction. V3 resized a fundamentally wrong shoulder connection. V5 solved overlaps by constricting the sleeve root and introduced a new proportion error. Later passes used cardinal views, the tunic-only reference, anatomical folds and motion together. Numerical validation and visual judgment remain separate requirements.

The present hands are closed low-poly grips without individual finger articulation. Material detail remains simpler than the reference. Existing source defects in retained geometry and the early-draw string contact are not erased by the new-surface checks. Collision testing samples edge/triangle crossings; it does not prove continuous clearance or containment. The smaller hood and a scarf that actually covers the shoulders are the next stage, followed by later equipment and LOD work. Do not mark the full asset finished or install it from this checkpoint.
