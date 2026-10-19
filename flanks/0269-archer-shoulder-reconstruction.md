# Archer shoulder reconstruction and arm deformation

Written by GPT-6.

The user rejected arms_v3 after identifying the horizontal shoulder extensions in refs/archer/archer_review2.png. The previous topology extruded sideways from the torso before descending into the arm. Reducing its width did not correct that structure. This iteration is assets_dev/archer/arms_v4; all preceding iteration folders remain unchanged.

## Geometry and reference

Rebuilt the torso-to-arm boundary around the shoulder cap and axillary folds. Removed the lateral connector rows. The collar-to-shoulder surface now slopes over the shoulder pivot, and the first sleeve section descends along the upper arm. Added an intermediate collar/shoulder row and a curved outer cap edge. Repositioned the posterior shoulder support after the nocking view exposed a pointed overhang.

The reference has loose wool sleeves ending near the elbow, gathered linen beneath, a shaped forearm guard and a continuous shoulder silhouette beneath its cape. The new sleeve uses different front, back, inner and outer profiles with separate width and depth. Forearm volume tapers toward the wrist. Clothed cross-sections are 129 by 127 mm at the upper sleeve, 119 by 111 mm at the sleeve hem, 107 by 91 mm at the proximal forearm and 53 by 41 mm at the wrist. These are model measurements, not dimensions inferred from perspective artwork.

Extended the left bracer through the lower forearm, ending just above the wrist, and added bound edges and stitches in its local atlas tile. Replaced the mismatched torso/sleeve UV mapping with cloth projection at a consistent scale. Existing retained atlas pixels remain identical. The current material detail is simpler than the reference illustration.

## Deformation

The source clips use the archer's existing 22-joint rig. No joint pivot or animation JSON changed. Elbow weights use position along the elbow bend bisector across an 180 mm region. This avoids folding the full arm width over a short hinge section. Wrist blending extends through the hand base across a 100 mm region and is shared by palm and thumb vertices. Shoulder weights spread through the upper chest and back, with separate front, top and rear support weights.

Rendered front, rear, right side and left side in neutral clay and cloth, then the existing start, reach, nock, pull and drawn poses. Visual inspection caught the posterior shoulder overhang after the intersection checks already passed; corrected its geometry and repeated both checks. The motion preview contains draw, raised aim and release at 24 fps for 87 frames, with all four views shown together. Inspected the complete ordered frame sequences from all four cameras and full-size key poses. This session did not provide a video playback tool, so no real-time playback or in-game visual approval is claimed. It uses the CPU reference for the game's skinning rather than Blender armature skinning.

## Exported checks

L0/L1/L2/L3: 2994/680/250/56 triangles. Height: 1.799999952 m. The new 820-triangle surface is one closed component. No open edges, nonmanifold edges, inconsistent winding, degenerate triangles or neutral self-crossings were detected. The 174 sampled poses across stand, ready, draw, aim_up, release and release_up have zero detected self-crossings in the entire generated surface and zero duplicated seam displacement. Minimum triangle-area ratio across those poses is about 0.116. These are sampled edge/triangle intersection checks, not continuous collision or containment proof.

The raw GLB passes glb_inspect.py. inspect_surfaces.py records per-part diagnostics; joined part boundaries are not independently capped. The original 22 joint nodes, pivots, retained L0 attributes, retained atlas pixels and L1-L3 geometry remain identical. Shoulder pivots are (+/-0.205, 1.435, 0), elbows (+/-0.290, 1.155, -0.010), and wrists (+/-0.350, 0.925, 0.130) metres. Complete part coverage and pivots are in validation.json and build_report.json.

Source SHA-256: d2ec53996146b9c7e0649b74a8a6a97dfd6e6aa570d04702c1bb6f188ec823c6.
Candidate SHA-256: d020960d16cc99a0690de8b6c1bcbf4cfa2b9a5199f6bbd227f8add2c711db16.

## Review boundary

This is the exposed-arm checkpoint, including basic deformation corrections because neutral-only inspection missed defects in earlier iterations. The full reference character is not finished. The smaller hood and full shoulder-covering cape still need their own fit pass; the original coif remains here. Hands are closed low-poly grips, without individual fingers. L1-L3 retain their source geometry. The back quiver, harness, sword/scabbard, bow stow and melee work in handoff 138 are separate stages.

Review files are review.html, clay_views.png, cloth_views.png, comparison.png, motion.mp4 and arms_review.blend. Bow and arrow are hidden in the arm inspection views but remain in the GLB. No public asset, engine code or animation clip changed. No commit was made. The iterations table records the v3 rejection and v4 review checkpoint.
