# Knight standing review and shoulder stiffness

Written by GPT-6.

## Approval status

The standing_v1 candidate is not approved. Gota explicitly confirmed this during review. The earlier comment that it looked good was an initial impression, not acceptance. Further inspection raised concern that the shoulders look held up or tense for an idle soldier casually carrying a sword. Keep the candidate and its measurements as work in progress; do not promote the model, integrate the pose or proceed to the ready-pose stage on the basis of that initial comment.

## Progress preserved

The standing candidate and reproducible scripts remain in `assets_dev/knight/standing_v1/`, preserved by private assets commit `1e2be3f` on that repository's main branch. The graphics tree remains on `feat/knight-stand`, with the five new source scripts under `tools/blender/knight/`. Devlog 0182 and note `work/notes/032-knight-standing-pose-2026-10-03.md` contain the build and renderer details.

The existing model was used at its original 1.80 m height and proportions. L0-L2 rest positions, joint coordinates, atlas pixels, part IDs and triangle counts were preserved. A separate pose table repositions the arms and sword. The candidate GLB also rotates the existing static L3 shield wedge while keeping every L3 vertex in part 0. The shipped `assets/units/knight.glb` was not replaced.

Measured posed widths are 0.832389 m at L0, 0.839465 m at L1/L2 and 0.699596 m at L3, with unchanged triangle counts of 2768/678/250/52. Furthest forward reach is 0.410989 m. The sword clears the ground by 0.143902 m. The formation, geometry and saved-scene checks described in devlog 0182 are engineering checks; they do not establish that the pose looks natural.

## New references inspected

Gota supplied `refs/animation/knight_standing/ref0.png`, `ref1.png` and `debug0.png` in the main tree. Both reference images are M2TW screenshots. `debug0.png` marks the shoulder contour and arm directions on the current candidate. These were compared with the saved candidate's front, side and three-quarter views and with the source sleeve geometry and pose code.

`ref1.png` gives the clearest visual comparison for an at-ease carry: the upper arms read as descending from the shoulders, with a less abrupt shoulder-to-arm silhouette. `ref0.png` contains several arm and weapon positions, so it is useful context rather than a single exact pose to copy. Differences in armor, camera and proportions limit direct silhouette matching. No joint measurements were inferred from these screenshots.

## Assessment

The current candidate is mechanically possible within the rig, but it does not convincingly read as a fully relaxed sword carry. The three-quarter camera emphasizes the shoulder shape, and the front view also shows the stiffness. The concern therefore cannot be dismissed as a camera effect.

The right upper arm still angles outward. Measured from the authored joint markers, its frontal angle from downward vertical is 21.750 degrees, compared with 27.187 degrees in the source rest mesh. Its full three-dimensional angle from downward vertical is 24.813 degrees. The elbow lies 89.669 mm outward and 52.514 mm behind the shoulder. The forearm then comes forward toward the hand. The hand target and backward/outward elbow pole in `Standing.author()` produce this arrangement; bringing the hand inward did not by itself make the whole arm hang loosely.

The source sleeve contributes to the appearance. `geometry_knight.py` constructs it as a thick tube with a rounded shoulder cap and a near-horizontal first section. In the standing deformation, that cap rotates with the upper arm about a fixed shoulder pivot. The body stays fixed; there is no separate clavicle or shoulder-girdle control or torso-to-sleeve skinning across that boundary. This can preserve a squared, raised-looking shoulder contour even when the arm's joint direction moves slightly inward.

The shoulder joint itself was not raised. Its right-side location remains (-0.224000, 1.435000, 0) m. The highest point in the original upper-sleeve vertex region actually moves down by about 4.945 mm on the sword side. The visual impression of tension comes from the outward/backward arm placement and shoulder surface shape, not evidence of an upward translation of the shoulder joint.

These observations identify both an authored-pose issue and a limitation of the shoulder surface treatment. They do not establish that the model needs a replacement skeleton. The current virtual shoulder, elbow and grip chain should be evaluated with a more relaxed upper-arm direction before deciding whether local sleeve geometry or deformation must change.

## Proposed next revision, not implemented

Start with the upper arm hanging more nearly downward and the elbow less far behind the shoulder; place the hand and sword to follow that arm rather than treating the current hand target as fixed. Check the shoulder-cap transition into the torso from front, back and side views. If the squared cap remains, correct that local surface or its deformation instead of hiding it with the camera or only moving the sword. Preserve scale and limb lengths, and recheck grip contact, blade clearance, shoulder overlap, width and all detail levels.

A new standing_v2 folder should preserve standing_v1 as the reviewed candidate. This turn only records the assessment. No pose, model, source script, engine code or iteration acceptance status was changed. No Blender run, game launch, build or commit was performed for this assessment. Standing_v1 remains unapproved.

## Follow-up: forearm and upward-forward sword carry

Gota added `debug1.png`, `debug2.png` and `ref2.png` under `refs/animation/knight_standing/`. The annotations request a more continuously descending arm with the sword carried diagonally upward and forward, using the M2TW knight screenshots as the visual target. The current candidate's elbow-back, forearm-forward arrangement and cuff treatment also need review. Correct the whole arm and grip together; changing only the blade angle would leave those concerns unresolved.

The point-down sword in standing_v1 was an authoring choice, not a requirement in handoff 085. Standing requires the complete figure to be no wider than 0.85 m and nothing more than about 0.5 m ahead of its centre. The separate ready-pose instruction to hold the sword up and back is not a standing-pose requirement. A diagonal upward-forward standing carry is compatible in principle, subject to its actual width and reach.

A read-only calculation using the original weapon vertices measures 0.865000 m from grip to tip along the blade axis. With the blade aimed forward, no lateral yaw, and the grip 0.055 m ahead of the body centre, its furthest forward extent is 0.666647 m at 45 degrees above horizontal and 0.487500 m at 60 degrees. With the grip at the body's forward/back centre, 55 degrees gives 0.496144 m. These are isolated weapon calculations, not an authored or validated replacement pose. A shallow diagonal could exceed the handoff limit; the reference-inspired carry need not do so.

The actual M2TW screenshots are the guide to the visible sword and grip. The extracted joint sheet has no blade-orientation drawing of its own, and the yellow annotation in debug2 lies beside the blue shield-arm chain; the red chain is the weapon arm. A single screenshot does not establish an exact three-dimensional blade angle. Adapt the intended carry to the knight's unchanged proportions and measure the resulting mesh.

No replacement pose was made during this follow-up. Standing_v1 remains unapproved.
