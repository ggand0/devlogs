# Archer dagger grip

Written by GPT-6.

2026-10-21. Gota provisionally approved the standing and ready poses, but rejected the dagger grip: the blade emerged through the four fingers instead of from the thumb/index side of a closed fist. This continues devlog 0279 in a new folder, `assets_dev/archer/melee_poses_v2/`. The preceding folder remains unchanged.

## Cause and correction

The first dagger was authored along +Z from the old arrow grip. The existing hand's finger mass also extends roughly forward from the wrist, so the dagger ran lengthwise through it. Checking the blade against the body did not catch this attachment error. The old inspection excluded hand/hilt contact as intentional without first establishing that the handle followed the correct anatomical axis.

The corrected handle follows the hand's width axis, across the closed palm and curled fingers. Its centre is derived from the actual hand geometry's grip centre, not the arrow nock: rest position (-0.360079, 0.898501, 0.195053) m, width axis (-0.090894, 0.927119, 0.363576). The guard is beyond the thumb/index side and the pommel beyond the little-finger side.

The right-wrist rotation is adjusted by the inverse knife-mount rotation. This preserves the dagger's world direction while putting the hand around the handle. Every other float in each saved pose sample is identical to the preceding JSON. The stance, shoulders, elbows, free hand, torso and head retain their reviewed values. Wrist-blended vertices naturally follow the changed wrist. No body or hand vertices, normals, UVs, weights or topology were edited, and no joints were added.

The blade remains 220 mm long. The model still has 2,994 / 680 / 250 / 56 triangles and the original 22-joint rig at 1.80 m rest height. The hand remains the existing closed-grip mesh without individual finger articulation. This corrects the dagger mount and wrist, not the hand's topology.

## Measurements and inspection

`check_grip.py` casts along the handle axis through the actual skinned hand and checks that it enters and exits the two side boundaries. Standing boundary offsets from the grip centre are -33.747 mm and +35.554 mm; ready is -33.747 mm and +36.139 mm. The handle extends from -55 to +45 mm, so both ends lie outside the fist, leaving roughly 9 mm on the index side before the guard. The skinned boundary changes by at most 1.338 mm from rest. An initial assertion that these intersections would remain exactly rigid failed, because the existing wrist skin also influences part of the hand. The final check measures the actual skinned contact in each pose rather than assuming rigidity.

All saved values outside `hand_r` are exactly equal to v1. Blade-axis difference is zero for standing and below 0.000001 degrees for ready. The corrected right forearm and hand have zero measured self-crossings in both poses. Blade/body, blade/right-arm, forearm/body and forearm/forearm checks all report zero crossings. Intentional handle/hand overlap is measured as the grip, not treated as an unintended collision.

Both clips pass the shared validator. Both soles remain grounded; the original 12 covered upper-thigh crossings remain unchanged, and exposed legs clear each other. Visible widths are 0.773 m standing and 0.647 m ready. The model's retained attributes, rig, pivots and L1-L3 data match the approved hood base. The GLB byte inspection passes the required channels, and the knife has no open, nonmanifold, inconsistent-winding or degenerate surfaces.

Inspected close-ups from front, both sides, rear, top, oblique and gameplay directions, then the corresponding full-body views. The close-up framing was widened to include the entire blade. The review begins with the grip details and includes the full-body standing and ready orbits. These remain static poses with a moving camera, not idle animation.

## Scope and next review

The grip revision remains uncommitted for review. V1's pose verdict is recorded separately from its rejected grip. A later approved v2 archive needs the two v1 pose JSON files that `build_poses.py` reads, without committing the rejected v1 model/review package wholesale.

The bow and held arrow remain hidden in the melee review. The original hip quiver remains while its rework is paused. Distance meshes still need matching melee equipment; L3's bow is merged into part 0. No draw, attack or transition was authored. No engine code or installed asset changed, and there was no game run or simulation hash test.

The approved hood source is still to be brought from `feat/archer-hood-source` into `feat/archer-rework` in `../flanks` later, as recorded in note 102. This task does not merge or switch either tree.

## Completed review media

Both updated orbit videos decode completely: 800 by 800 pixels, 576 frames at 24 fps, 24 seconds each. All local links resolve. An isolated headless browser played each through the entire two-direction orbit and into the next loop. Ordered browser captures from both directions were inspected for full-body framing and the corrected hand silhouette. Grip close-ups were inspected separately at full resolution. This is offline pose review, not an in-game test.

No commits or pushes were made. Devlogs were backed up to `/data/ggando/flanks/devlogs/`.

## Approval and next work

Gota approved the dagger grip and both poses on 2026-10-21 and requested the progress log and commits. The sword draw is skipped rather than revised. Next is a new back-quiver iteration based on the M2TW archer placement, fitted close to the body with no visible harness. The previous rejected quiver and draw folders remain unchanged.

## Commits and build reproduction

Private archive: `df34e9a`, including the two v1 pose JSON dependencies. Public source and pose JSON: `8aac4a9` on `feat/archer-hood-source`. The public build reproduces the approved GLB byte for byte (`5183613dd544e2efbf2f5a5d7ac6d1233798df6bd15f83dfb5638437dec92e9a`); raw GLB inspection passes. Images and videos in the private archive use LFS. No final unit GLB was installed. Integration remains for `feat/archer-rework` in the main tree later.
