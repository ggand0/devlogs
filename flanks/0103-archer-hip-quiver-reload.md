# Archer hip-quiver reload

Written by GPT-6 Astra.

2026-09-24. Gota accepted `assets_dev/archer/release_v5/archer_release.mp4` with "release motion looks good" and requested the reload. Recorded that acceptance in devlog 0102. Authored the reload only; it is awaiting playback review. The approved raise and release tables remain exactly equal to their saved baseline, and their existing review videos are intact.

## Motion and preview

The 5.8-second clip begins at the approved follow-through endpoint. The torso straightens and the bow lowers to a vertical hold in front of the body. The drawing hand first clears the shoulder, then reaches around the right side to the actual hip-quiver arrow tails. It lifts to extract an arrow, carries it outside the shoulder, brings it forward and nocks it at 3.8 seconds. The final two seconds bring both hands to the approved horizontal drawn-bow pose. The bowstring follows the nock during this final motion. Release plus reload occupies the shortest 6.4-second recovery interval from the incoming brief.

The first reach trajectory passed too close to the shoulder and folded the elbow beyond 150 degrees. Routed the hand forward of the shoulder before descending. A first arrow-transfer path intersected the upper arm when the hand lowered; moved the transfer outside the shoulder and briefly tipped the extracted arrow back by 12 degrees. The final return also follows a shallow forward arc to clear the hood/cape instead of taking a straight path through the torso envelope. These changes address the moving geometry, without changing the accepted clips or relaxing the elbow/clearance checks.

Preview: `assets_dev/archer/reload_v6/archer_reload.mp4`. Its `review.html` provides slow motion, frame stepping and replay reload. The sequence has 265 frames at 30 fps, viewed from oblique and side cameras. Reload runs from 0.40 to 6.20 seconds, followed by a 0.40-second horizontal hold and the approved raise beginning at 6.60 seconds. `archer_reload_review.blend` contains the complete sampled animation. Full frames are in `frames_oblique/` and `frames_side/`.

The visible held arrow is a separate part. It appears 1.65 seconds into reload, during extraction, then follows the drawing hand until attachment to the string. The six bundled arrows remain decorative quiver geometry; this pass does not remove one from that stock. Fingers remain part of the rigid hand rather than individually animated joints.

## Source and format

Source remains in `tools/blender/archer/`. The reload adds a 257-row joint table, while raise and release retain 65 rows each. Each joint row still has the same 30 values. `archer.raise.json` now uses format version 3 and adds `arrow_samples.reload`: 257 rows containing a torso-local nock position, XYZW orientation and string-attachment weight. Before attachment, the arrow follows the hand. During nocking, its free transform blends to the existing string-derived transform. Once attached, it uses the same nock/arrow calculation as the accepted raise. Quaternion interpolation follows the shortest path and normalizes the result. Visibility follows the explicit event time, not a blend between visibility samples.

`sample()` uses each table's row count instead of assuming 65. The preview and scene checker accept `--reload`; the motion checker covers all clips and accepts `--baseline` to compare every clip in a prior export. Reproduction commands and the extended format are documented in the README. No runtime source or renderer code was edited. Claude will need to support version 3, clip-specific row counts, the companion arrow table and discrete visibility events when integrating.

Before editing, saved the approved source to `reload_v6/accepted_release_source/` and the previous table to `accepted_raise_release.json`. The preview GLB is byte-identical to the approved release/raise mesh, SHA-256 `cf9c79d95905c7615981970d3916e9bfe6362fe9d51740d8aaa26ded260c0812`. Geometry and the atlas were reused without rebuilding; no mesh changes were needed.

## Validation

Ran the GLB byte inspector and a 1,001-pose sweep per clip, 3,003 poses total. L0 remains 2,987 triangles, height 1.7999999523 m, with the existing part IDs, pivots, VEC4 color and both UV attributes. Both approved clip tables compare exactly against the baseline. Maximum posed-vertex boundary error is zero for release to reload and 2.221e-16 m for reload to raise.

The drawing hand stays at least 4.306 mm outside the convex torso envelope. Draw-elbow flexion spans 60.426–145.827 degrees across the clips; bow-elbow flexion spans 26.305–102.657 degrees. The shared hinge and straight-wrist checks pass. All measured joint-cover insets remain positive: draw shoulder/elbow/wrist 15.314/13.138/8.880 mm; bow shoulder/elbow/wrist 15.418/12.832/8.880 mm.

During reload, sampled shaft centerline clearance outside convex part envelopes is at least 62.378 mm from the draw upper arm, 5.831 mm from its forearm, 7.845 mm from the bow forearm, 24.125 mm from the head and 41.683 mm from the torso. The check starts 80 mm from the arrow nock to exclude the intentional grip/fletching region and requires more than 3 mm of clearance for the shaft. Maximum free-arrow grip discrepancy is 0.898 mm, and maximum finger/string discrepancy during nocked motion is 0.0965 mm.

Every saved Blender playback frame agrees with the sampled geometry within 7.153e-7 m. Rendered the whole sequence and inspected sequential frames through the reach, extraction, transfer, nocking and return. The user's video review is still required. These checks do not establish complete self-collision or surface acceptance, and no in-game playback was performed.

Remaining stages after reload review: archer melee, then cheers for archer, knight, man-at-arms and spearman, one kind and review stage at a time. Full LODs and runtime integration remain separate. No installed assets, commits, staging or pushes were made by this work. The concurrent knight asset change was left alone.
