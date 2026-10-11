# Knight mid stab approval and implementation

Written by GPT-6 Astra.

2026-10-11. Gota approved stab_v1: "looks great, approved." He requested the acceptance commit, a new implementation devlog, and a handoff for Claude to integrate it on the current branch. He asked about the charge's scope; charge authoring was not started in this approval turn.

## Accepted artifact and commits

The reviewed animation and media are unchanged from private assets commit `c8198b19379a423c545f3d7febbca48d55657ec2` (Add the knight mid stab v1). Acceptance and implementation notes are committed on private assets main as `e6fe30f67857fe0d8923626056e7e77cbabfebe2` (Record the knight mid stab acceptance). That commit changes only the iteration README, its checksum entry, and the shared iteration row. No push.

Source: `assets_dev/knight/stab_v1/knight.stab.json`. SHA-256: `d4aa62e43dda56e3ba0083d5d57f4d5b0482a36c53a8e48038ea86947cc64f06`. Model: `assets_dev/knight/skeleton_v1/knight.glb`, SHA-256 `2df8514e28bdf73941bfdc8d6c354b78a154be6e8809fa3ca7a387b4c9c8acc8`. Ready: `assets_dev/knight/skeleton_v1/knight.ready.json`, SHA-256 `e78053c67b192fef7ccca0c7f9b5abcd73de4b75b2be2c23c8e85d8fd33d7f1b`.

Format flanks-clip v1; skeleton knight; 19 joints; root offset XYZ plus 19 XYZW parent-relative rotations, 79 floats per sample. There are 21 samples at 20 Hz, duration 1.00 s, contact 0.40 s, no loop and travel `[0,0,0]`. Both endpoints exactly equal the canonical ready sample. Contact tip `(0.192423, 1.247297, 1.598957)` m, +X left, +Y up, +Z forward. This is an ordinary style-0 stab, not a charge attack.

## Implementation

`build_stab.py` constructs a `Stab` around the shared `Glb`, skeleton and `Pose` APIs. It loads ready, computes its joint origins/rotations, derives each ready knee pole from the hip-knee-ankle plane, and derives the shield normal from the shield-bound vertices. No model or skin mutation occurs.

Body controls use cubic Hermite curves at 0.00, 0.20, 0.40, 0.50, 0.72 and 1.00 seconds. Most control tangents are zero; the withdrawal key at 0.72 s has the centered tangent so the return does not stop there. Pelvis and chest have separate yaw and lean. The pelvis lowers from the ready 0.085 m drop to 0.160 m at contact; yaw moves from -55 to +5 degrees, chest yaw from -55 to +35 degrees and chest lean from 10 to 30 degrees. The contact root shift is -0.025 m sideways and +0.060 m forward. These are asset-local displacements, not unit travel. The head keeps facing forward.

The front ankle and foot rotation retain ready. The rear toe anchor is computed once from ready using local offset `(0,-0.091,0.258)` m. A smooth weight rotates rear-foot yaw -80 to -15 degrees and pitch 0 to 14 degrees. The ankle is solved as anchor minus rotated toe offset, so the heel lifts around the toe. Both legs use the shared leg IK with evolving knee poles, and unreachable targets raise an error. No root translation is applied outside the pose.

The sword arm is solved by two-bone IK at independently authored keys. Upper/forearm rest lengths are measured from the skeleton. Joint frames use the minimum alignment from the ready bone frames. The final motion uses a model-space upper-arm rotation spline and a parent-relative forearm rotation spline. Their keys include an extra 0.30 s clearance pose: hand `(-0.43,1.13,0.22)` m and elbow target `(-0.50,1.02,-0.04)` m. This steers the hand outside the torso; direct world-space pole interpolation had flipped the elbow between otherwise valid keys. The final path avoids that flip and all sampled equipment crossings.

The sword's direction uses its own smooth controls. Its model-space orientation is transported from ready by the minimum alignment to the authored blade direction. The hand and weapon receive that same orientation, keeping the modeled grip together. Contact hand target is `(-0.24,1.23,0.85)` m; the blade direction control is `(0.50,0.02,0.866)` before normalization. The blade reaches about 1.60 m forward without the old low aim.

Shield upper-arm and forearm directions are authored separately. A turn about the forearm axis gives the closest attainable board normal under its existing mount. This cannot match both the reference forearm and board normal simultaneously. The shield follows its forearm and is not independently detached to conceal that limit.

`Pose.sample()` bakes coat helpers. The writer emits 21 samples, marks contact, then replaces first and last sample with the canonical ready arrays exactly. The JSON already contains the final motion; the game needs its existing quaternion clip player, not these authoring solvers.

## Validation and appearance

Devlog 0225 contains the full measurements and construction history. The shared validator passes, peak turn 1239.945 degrees/s. The build reproduces the reviewed JSON byte-for-byte. Four-level checks sample 201 interpolated times; equipment checks sample L0–L2 at those times. No blade/arm, weapon/body, shield/body or weapon/shield crossings. Maximum width across levels 1.305610 m. Ground interpolation penetration at most 2.244 mm; front sole and rear toe drift about 3.3 mm. All legs remain reachable, minimum margin 33.319 mm.

Reference checks use exactly four events against each of the knight and militia: ready at frame 0, wind-up at 8, contact at 14 and ready end at 28. The timing and intermediate curves are ours. All derived reference images/tables remain in ignored local_reference. The shield-mount constraint, fixed front-foot facing, rear-toe pivot and shorter limbs account for documented OVER rows. Approval accepts the presented appearance; it does not erase those measurements.

Coat/thigh intersections and the shield sleeve's missing elbow loop remain. L0 has small upper-thigh intersections under the coat during recovery. Side neighbors clear at 1.4 m in sampled arrangements and can intersect at tighter spacing. The thrust reaches into the rank ahead. These are integration constraints, not changes to simulation spacing.

The review includes play-speed and half-speed videos, two-direction orbit, seven views, both references and all four detail levels. Browser playback, loop advancement, link/image checks and full MP4 decoding passed in the authoring turn. That evidence is unchanged, so no render/build/game rerun was needed for this documentation-only approval update. SHA-256 checks confirm the accepted clip and review media are unchanged.

## Current branch and integration handoff

The graphics tree is on `feat/skeleton-clips` at `c8bd74cb0d131ff81fcb233bc7135dfac122b5a4` at this inspection. Claude owns the player work there. Handoff 105 gives the integration steps and verified current code locations. The current loader lists seven clips through cut; the Rust capacity is eight. The shader plays the cut from a clip on L0/L1 and still applies the procedural arm stab to other blows. The new clip is not wired yet.

Follow the existing coordinated integration arrangement of handoff 094: promote the clip and portable authoring/check scripts together with Claude's player work. The shared skeleton modules have not been promoted under tools/blender, so blindly copying this iteration-relative build script there would not be a reproducible public build. Handoff 105 names that dependency and requires explicit paths plus a byte-identical rebuild. Shared tools, src files and the current graphics branch were not changed in this approval turn.

## Next animation

Handoff 104 item 2 is `knight.charge.json`, a looping whole-body charging stride, not a single pose. Author it on a treadmill and record cycle travel; the player advances phase by ground covered at the charging speed. Both feet may be airborne during the run. The validator's shuffle support requirement needs an explicit run-specific treatment, not a hidden tolerance change.

Confirmed charge, ready_to_charge, run_to_charge and charge_to_ready GIFs/sheets for both dismounted_feudal_knights and swordsmen_militia. The charge cycle is the next review item. Entry/exit references guide transitions but do not mean all transition clips are part of that one deliverable. The charging strike is item 3, a separate `knight.charge_attack.json` with 0.4 s to contact and 0.6 s to ready; its reference is still absent from the render folders and ATTACKS list at inspection. Claude must prepare that reference for both units before it is authored. Optional overhead remains a later decision.
