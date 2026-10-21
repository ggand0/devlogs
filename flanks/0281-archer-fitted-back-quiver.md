# Archer fitted back quiver
Written by GPT-6.

## Request and approved work

Gota approved `melee_poses_v2`: the standing and low guard poses and the corrected right-hand dagger grip. The private archive is `df34e9a`; public dagger build source and the two pose JSON files are `8aac4a9` on `feat/archer-hood-source`. Devlog 0280 records the byte-identical public rebuild. Integration belongs in `feat/archer-rework` in `../flanks` later; no merge or main-tree source edit was performed.

The sword draw is explicitly skipped. `sword_draw_v1` and `quiver_v1` remain rejected and untouched on disk. The new quiver should fit directly against the back without rendering a harness. The placement reference is `refs/animation/m2tw_model_refs/peasant_archers_melee/stand_a_idle.gif`, inspected from rear and both sides. Reference media remain outside the repositories.

## Observations and change

The reference quiver runs from the left waist toward the right shoulder. Its upper opening is behind the shoulder, and the arrow ends sit beside the hood below the top of the head. The previous hip quiver hangs too low for that silhouette. The earlier back-quiver attempt placed its opening farther back and higher and added an unwanted harness.

New iteration: `assets_dev/archer/quiver_v2/`. It starts from the approved dagger model. The old 290-triangle hip quiver is removed; the dagger, sheath, body, arms, hood, scarf and their attributes are preserved. The replacement consists of a closed 108-triangle leather case and five closed shafts with fletchings, 120 triangles combined. The case has a tapered oval cross-section, a rolled opening and inner lining. Its centreline curves slightly over the back of the scarf. There is no harness mesh, chest belt or extra attachment joint.

At rest, the base is `(0.140, 0.980, -0.150)` m and the opening is `(-0.205, 1.480, -0.225)` m. Positive X is the soldier's left. The endpoint distance is 612 mm; the outer lip is 106 mm wide and 85 mm deep. The arrow ends project 171–200 mm beyond the opening. The entire quiver follows the existing chest joint, on torso part 19 and its existing pivot height.

## Fit correction within this iteration

Back-surface ray measurements informed the initial placement. The first cardinal render showed the intended silhouette, but the edge-crossing check found scarf-hem intersections at the exposed side of the case. Simply checking the arms and head would have missed those. Testing the same rigid mount 25 mm rearward cleared the exposed back-facing case panels in all sampled poses while retaining contact on the concealed front side. The final renders use that corrected position.

## Validation

- 2,932 / 680 / 250 / 56 triangles at L0–L3; rest height 1.800 m; 22 joints.
- Retained vertex attributes, joint data, nodes and pivots match the approved source exactly. Lower-detail meshes also remain exact copies of the source.
- The new 228-triangle quiver has zero open edges, nonmanifold edges, inconsistent edge winding or degenerate triangles after welding.
- 177 poses sampled across rest, both melee poses, ready, draw, upward aim and both release clips. No quiver crossings with the head, either arm, bow, held arrow or exposed back-facing case panels.
- Concealed front-side clothing contact remains intentional: up to 59 edge crossings during motion. The nearest sampled case-to-clothing distances are 1.25 mm at rest, 0.64 mm standing and 3.16 mm in melee ready. Those are local contact measurements, not a claim that the whole case touches the body. Edge-crossing tests do not prove absence of containment.
- Raw GLB channel inspection is saved in `glb_inspection.txt`; the detailed geometric and motion report is `validation.json`.

## Review and remaining stages

Full-body cardinal and elevated views, quiver close-ups and 24-second full-body orbits cover standing and melee ready. A third orbit runs the existing draw, upward aim and release motion with a review-only blend back to the start. Bow parts are hidden in melee; dagger triangles are hidden for the shooting check. No animation clip is altered by this iteration.

The existing arrow retrieval still reaches to the hip. Retargeting it to the new opening is the next animation stage after this placement review. Matching L1–L3 quiver silhouettes is also separate; their present meshes retain the old hip quiver. The complete LOD/equipment acceptance checklist has not passed, so no final `assets/units/archer.glb` is installed. This iteration is pending Gota's review and is not committed.

## Completed review media

All three MP4 files decode successfully: 800 × 800, 24 fps, 576 frames, 24 seconds each. Each played through a complete loop in an isolated headless browser, with eight ordered captures per video. Cardinal views, quiver close-ups, the 16-frame shooting sequence and browser capture sheets were inspected. The shooting camera was widened to retain the whole raised bow; the melee cameras retain the closer full-body framing. `review_validation.json` records playback and media checksums. All render reports match the corrected model checksum `7bfc195633ce52867e9ff519570ecb4c9860aa55ffcc7999b480bd8c93df2e42`.

Review entry point: `assets_dev/archer/quiver_v2/review.html`. The compact rear/left/right/front comparison is `cardinal_views.jpg`. Final status: pending review, uncommitted. No gameplay or rendering code in src was changed, so no sim hash check was run.
