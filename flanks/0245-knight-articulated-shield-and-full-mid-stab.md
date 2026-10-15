# Articulated knight shield and full mid-stab reference motion

Written by Codex (configured model: GPT-6 Astra).

2026-10-13. Gota rejected block_v2's fixed shield mount and its changed existing poses. He explicitly permits new joints and requires one model for all animations. He then reiterated that the block must look like defend_mid_stab.gif and defend_mid_stab_sheet.png, and requested the result quickly. The latest full-reference request supersedes the earlier held-guard interpretation for this visual candidate.

## Rig and original animations

Work is in assets_dev/knight/block_v3. A new forearm_twist_l joint separates pronation from elbow flexion. Hand and shield are separate children of the twist joint with a common grip origin. Rest geometry, original joint origins, UVs, colors and triangles remain unchanged. Counts are 2,768 / 678 / 250 / 52. The new rig has 20 joints.

All ten existing clip tables preserve their original named joint channels exactly, adding only an identity twist channel. Dense 100 Hz comparisons of all four detail levels verify original vertex positions, normals and joint transforms to numerical precision. The new rig therefore does not need v2's rewritten ready, carry or attack poses. The shipped model and clips are not changed.

An attempted blended proximal strap moved through the board during recoil. That deformation is removed before review; the original rigid strap binding remains. Independent shield tilt does not establish that the leather straps maintain correct forearm contact. Hand and strap contact remain a close-review limitation and are not covered by the body/equipment clearance checks. Do not present a common grip origin as proof of correct physical grip surfaces.

## Animation and comparison

The entire 17-frame reference GIF was inspected as ordered frames, along with its seven-view sheet. Four reference events were measured: start, frame 4 guard, frame 9 recoil, and return. Three independently authored pose targets and eased transitions create the new motion. Timing is 0.80 seconds at 20 Hz, with raise at 0.20, recoil near 0.40–0.50, and recovery by 0.80. The new start/end pose follows the reference instead of inheriting the accepted ready pose.

The guard comparison is within all 18 tolerances (2.8 degrees angular RMS). Recoil is 4.7 degrees RMS with one row above tolerance: shield-side shin 14.3 degrees. The leg IK holds the model's heel positions; the proportions and foot arrangement differ from M2TW. These comparisons do not imply an exact visual reproduction.

The shared saved-clip validator and dense motion checks are recorded beside the scripts. The CPU-skin renderer supplies seven views. review.html pairs the actual reference and new motion in synchronized full-speed and half-speed videos. The reference and paired videos remain local and ignored. The browser check decodes each video and captures its full playback cycle and wraparound. This is offline verification, not an in-game test.

history.html lists the preserved versions: A is rejected v1; B is the earlier static body/hinge diagnostic; C is rejected v2. B never had a completed animation. Earlier promises to copy the full recoil while also claiming the handoff's hold timing were contradictory; the full-reference preview now states its timing and endpoint differences explicitly.

## Status

Visual review candidate at http://127.0.0.1:8767/block_v3/review.html. No commit, push, source edit or shipped asset replacement. Existing v1 and v2 files are preserved. Integration must use the 20-joint model/clip package and resolve entry blending and the sim's former 0.90-second timer. Parry remains a later separately reviewed item.

Final candidate checks: 161 interpolated poses, exact shared start/end sample, left elbow plane error below 0.000008 degrees, wrist swing 35.11 degrees, heel drift 4.085 mm. Saved-pose width 1.20678 m. Blade/arm, weapon/body, shield/body and shield/weapon checks are clear at 81 phases on L0–L2. Shield/arm contact is separate: crossings exist in the original grip and increase during the new motion, so equipment clearance must not be described as all surfaces clear. Complete synchronized browser playback and loop captures were inspected as ordered images.

## Provisional acceptance

Gota accepts block_v3 for now and requests its commit on 2026-10-13. He will evaluate it in-game and return with any problems. This accepts the presented motion without claiming the known grip/strap contact limitation is resolved. He explicitly requires an orbit view from the next animation onward; include the full clockwise and counterclockwise orbit in the parry review. The next reference is defend_mid_slashlr, not defend_mid_slashrl.

The approved package is committed as c069a14 on the private assets repository main. Review media are LFS pointers; reference-derived media and build output are excluded. Handoff 116 records the complete model/clip package and the timing/entry-blend differences for in-game evaluation.

## Final common model for block and parry

On 2026-10-15 Gota approves parry_v2 and its corrected shield mount. Use parry_v2/compatible/knight.glb with the complete matching clip package for both actions. The block clip remains byte-for-byte unchanged; only shield geometry and strap normals differ from this iteration's model. The final package is archived in 97843937e2c6f5dc8e58efaf88d14756b4a18ad8. Handoff 118 supersedes handoff 116's model path and records that knight block and parry assets are complete; player integration and in-game checks remain separate.
