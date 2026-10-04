# Knight forward and backward shuffle v1

Written by GPT-6 Astra.

Created the next shuffle pair in `assets_dev/knight/shuffle_fb_v1/`: `knight.shuffle_forward.json` and `knight.shuffle_backward.json`. Each travels 0.60 asset metres in 0.75 seconds, at 0.80 m/s, with sixteen samples at 20 Hz. Both endpoints exactly equal the saved `skeleton_v1/knight.ready.json`. Gota approved both clips on 2026-10-04 after the first visual review. This records offline animation approval; player integration has not been tested here. Standing, ready, side shuffle v2, the shared model/skin/tools and engine code remain unchanged.

## References and authored motion

Inspected both units' full `shuffle_forward` and `shuffle_backward` sheets and eleven ordered frames from each corresponding GIF: swordsmen militia for visible limbs and dismounted feudal knights for the equipped silhouette. Four key reference poses per direction, frames 0, 9, 17 and 30, confirm the support order. Forward moves the rear right foot first; backward moves the front left foot first. In both cases the foot trailing the direction of travel closes first, then the leading foot moves.

The independent step closes the accepted 0.620956 m fore/aft ankle stagger by 0.60 m, leaving about 0.021116 m instead of passing the feet as the longer M2TW stride does. Lateral tracks retain ready's roughly 0.304 m separation and toe directions. Forward lift targets are 65/55 mm, backward 45/45 mm. Forward raises the pelvis 18 mm while backward lowers it 18 mm. A one-degree chest pitch and 10 mm lateral shift envelope supply small body motion; the ready guard and coupled hand/weapon orientation remain intact. This is a new authoring script, with no dependency on legacy side-shuffle angle tables and no copied reference curves.

The 0.60 m cycle was chosen to nearly close the actual ready stagger while retaining the existing step pace. Handoff 085 imposes no forward/backward cycle-distance target. The compact, non-crossing footwork is an explicit variation for Gota's look, not a claim that it matches every reference angle.

## Measurement and contact

`reference_checks.py` compares our ready, trailing-foot swing, leading-foot swing and ready return at 0.00, 0.20, 0.55 and 0.75 seconds to the four reference frames for both units. Sixteen comparisons, tables and front/side/top overlays are under ignored `local_reference/`. Each OVER row has a measured endpoint baseline and a reason for its retained difference. No additional reference poses were sampled for optimization.

The retained guard, fixed toes, smaller stride and lack of foot passing leave substantial differences. The forward leading-foot pose against militia reports 32.6-degree angular RMS and 15 of 18 rows over tolerance; the backward trailing-foot pose has 16 of 18 over. The reference turns its torso and feet through the step, while this candidate retains ready's facing. These are explicit retained variations accepted in the visual review, not a passed reference score.

The shared validator passes both saved clips. The iteration checks evaluate 301 interpolated times on all four levels and contact-boundary times separately, using the game's CPU skin deformation. At least one foot stays grounded. Leg targets remain reachable without IK clamping; saved-sample ankle error is below 0.000124 mm. Interpolation between the 20 Hz samples deviates from the analytic swing target by up to 10.05 mm, a different quantity from the planted-sole drift below.

| Measurement | Forward | Backward |
|---|---:|---:|
| Maximum planted-sole world drift | 1.3495 mm | 1.0303 mm |
| Maximum ground penetration | 0.5252 mm | 0.5771 mm |
| Maximum width | 0.949324 m | 0.949324 m |
| Maximum forward reach | 0.680175 m | 0.686040 m |
| Maximum rear reach across levels | 0.348727 m | 0.348694 m |
| Minimum remaining leg reach | 20.362 mm | 21.436 mm |
| Largest interpolated foot rotation difference from ready | 0.1804 degrees | 0.1371 degrees |

Exact endpoint position does not imply equal velocities. The additional loop check reports a maximum vertex-velocity difference of about 0.172 m/s forward and 0.092 m/s backward at the boundary, and sole speeds up to 0.047 m/s around it. These are finite differences on the saved 20 Hz interpolation, including numerical precision effects. They are exposed in `contact_validation.json`; no claim of exact velocity continuity is made. Arbitrary mid-cycle stopping still needs a contact-aware player transition.

## Surfaces and available space

At 31 phases on L0–L2, blade/arm, weapon/body, shield/body and weapon/shield crossings are zero. L0 legs do not intersect. L1 thighs have 12 edge crossings in ready and up to 16 during either clip, at the upper attachments. L2 adds four crossings during the close stance, absent in ready. Localized crossing heights are about 0.776–0.844 m forward at L1, 0.816–0.838 m at L2; backward 0.753–0.808 m at L1, 0.791–0.799 m at L2. These are recorded surface limitations; the coat does not establish that every crossing is concealed.

Coat/leg ready/maximum crossing counts are L0 73/97 forward and 73/101 backward; L1 122/128 and 122/131; L2 64/66 in both. Edge counts do not measure visible penetration depth. The review includes close views of L0 through L3 at ready, near minimum stagger and the second step. Shared geometry and helper weights were not altered to hide the counts.

Neighbor checks cover four spacings (0.9, 1.0, 1.05 and 1.4 m), cardinal offsets, same and half-cycle phases, and actual root travel toward/away from stationary ready and standing soldiers. At L0, fixed relative offsets are clear at 1.0 m and above; 0.9 m retains ready-neighbor intersections. A complete forward step toward a standing soldier can intersect even from 1.4 m starting spacing (27 sampled edge crossings maximum), while a ready neighbor at that spacing clears in both directions. Backward toward standing also clears at 1.4 m. Both directions intersect neighbors at the tighter spacings when closing the full 0.60 m gap. The JSON holds each phase and detail level; these are sampled checks, not continuous collision guarantees or crowd avoidance.

## Review and workflow experience

The review page contains both fixed-grid movement views, seven-view sheets, five-view GIFs, both local reference sets with comparisons/exceptions, and clockwise/counterclockwise moving orbits. Each orbit is 24 seconds at 24 fps, with animation time always advancing. Standard glTF inspection export is separate from CPU-skin review rendering. Raw inspection and attribute comparisons confirm that triangles (2,768 / 678 / 250 / 52), height, atlas, channels, pivots and skin are unchanged.

The first world-space videos exposed a camera issue in timed browser playback captures: the side-step camera scale clipped a boot in forward travel and the helmet in a gameplay view. The fix computes a single orthographic width from all projected vertices across both directions, both views and full root travel, with a 12% margin. It keeps the review scale comparable and does not alter the animation. The four world videos were re-rendered after that correction.

This is the first new animation authored with the documented workflow. Checking the support order before writing paths avoided the conceptual mistake from side shuffle v2. The measurement tool quantified large structural differences, especially ankle crossing and torso/toe turns; it did not automatically improve the animation. No pose revision was needed to pass the contact checks, and Gota accepted both clips at the first visual review. This pair took one authored candidate and one review-camera correction, found visually after successful video decoding. That is a useful result for this pair, not proof that every new animation will need only one review. Numeric checks and motion inspection continue to answer different questions.

`docs/internal/006-unit-animation-workflow.md` is updated with these findings. The guide now records the first-review approval and keeps player integration separate. Source/clip hashes, all reports and scripts remain with the iteration; local reference output, raw frame sequences and build binaries remain outside commits. Game integration is described separately in note 041.

## Final verification and archive

Saved the iteration and its private README row in assets_dev main commit `072dc90` (Add the knight forward and backward shuffles). The commit contains 37 files, with all 11 review binaries verified through Git LFS. The 35-file checksum manifest matches the reviewed package. Local reference data, raw frames, atlas, inspection GLB and documentation remain outside the commit. Nothing was pushed. The game worktree remains clean; the other session's cut folder and unrelated private assets are untouched.

The final page resolves all 63 local links. All six MP4s decode fully: four world-space videos at 720×540, 48 frames and 24 fps; two orbits at 768×768, 576 frames and 24 fps. Separate headless Firefox playback observed a loop for both directions, with 83 and 85 decoded frames respectively. Timed playback captures, ordered poses, lower-level images, representative orbit angles and corrected extreme-travel framing were inspected. The fitted world camera is 3.7 m wide, against a measured minimum of 3.242 m. This records browser playback and visual capture inspection, not an in-game test. No desktop input was sent.

The pair is visually approved. Script promotion and player integration remain coordinated follow-up work.

## Visual approval

On 2026-10-04 Gota approved both directions: "Both look fine and good enough to me. Approved." The accepted animation package is unchanged from private assets commit `072dc90`. Verified all 35 checksum entries before recording approval; no animation, model, measurement or review-media edits were needed. Private assets commit `aa09a5d` updates the iteration row to Motion accepted. No push was made.

This accepts the presented footwork and retained reference variations. The documented skin intersections, formation-space requirements and player transition requirements remain visible for integration. The integration note and workflow guide now record approval. Documentation stays uncommitted and devlogs are backed up at session end.
