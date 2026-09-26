# Knight diagonal sword slash

Written by GPT-6 Astra.

2026-09-24. Gota requested an M2TW-like diagonal sword slash, knight first, using `resources/knight/knight_diagonal-slash_ref.png`. The motion should be reusable for MAA and future sword-bearing units. Archer ranged animation work is complete for now, and its celebration rework remains deferred.

Authored the first knight preview in `assets_dev/knight/diagonal_slash_v1/`. `knight_diagonal_slash.mp4` and `review.html` show an oblique and a 50-degree gameplay-camera view. The 108-frame, 30 fps preview contains one normal-speed cut followed by one half-speed cut. The clip itself lasts 0.60 seconds: low ready, high right guard, diagonal cut toward the left hip, low follow-through and recovery. Contact is at 0.27 seconds. The feet stay planted while the torso twists about 9 degrees each way and the shield turns clear of the cut. This is an asset-side preview awaiting review, not a runtime change.

The reusable source is `tools/blender/melee/sword_slash.py`. It reads shoulder, elbow, grip and shield markers, and scales its shared hand trajectory by the measured total arm length. It solves the two-bone arm during authoring, then exports 121 samples with 14 channels: local shoulder, elbow and grip XYZW quaternions, torso yaw and shield yaw. The connected sleeve blends around the elbow and grip; the hand and sword share the grip rotation. The runtime need not solve IK. The adapter matches the current knight/MAA connected-sleeve convention. Independently tagged forearm/hand rigs need an adapter, and a long spear needs a different weapon profile rather than blindly reusing the sword's cut.

Measured knight arm length is 0.493244 m. A data-only retarget check creates all 121 finite poses for the installed MAA markers at 0.482727 m arm length. No MAA or spearman asset or preview was authored in this stage.

Supporting source: `tools/blender/knight/prepare_slash.py`, `check_slash.py`, and `review_slash.py`. Commands and the clip contract are in `tools/blender/melee/README.md`. The early scripts left under the development output are working snapshots; the promoted tools are authoritative.

The existing cuff exposed diamond-shaped mail intersections during the raised pose. Added a closed 12-sided leather overlap using the existing atlas, with 68 triangles. The first spherical overlap did not address the intersection and was replaced in the build source by a tapered cuff. Final preview mesh: L0 2,836 triangles, L2 250 triangles, height 1.7999999523 m. The installed knight asset is untouched. L2 is preserved from the input and was not reviewed through this new animation.

GLB byte inspection confirms COLOR_0 VEC4, UV0/UV1, the existing part IDs and pivot nodes. Weapon-arm part 1 has 340 triangles with zero open, nonmanifold or same-winding edges. Other inherited body surface exceptions remain; see the full surface report. This is not a claim of full final-asset acceptance.

The 241-pose check passes: exact grip attachment, elbow hinge error below 1.38e-15, sword edge-length error below 4.8e-16 m, exact loop closure and unchanged feet. Elbow flexion ranges from 55.25 to 114.57 degrees. Maximum authored grip rotation relative to the forearm is 64.73 degrees. No sword surface intersections with body, shield or legs were found in the sampled sweep. All saved preview frames match their expected deformation within 1.85e-6 m. Inspected sequential poses in both views and the raised cuff close-up. These checks do not substitute for Gota's full playback review.

Knight marker coordinates in glTF metres: shoulder (-0.224, 1.435, 0), elbow (-0.337, 1.215, 0.012), grip (-0.400, 1.084, 0.210), shield shoulder (0.224, 1.435, 0). `knight.slash.json` records the sampled table and exact markers. The new 3D clip format is separate from the engine's existing pitch-only sword attacks.

No `src/`, renderer or installed asset changes, staging, commits or pushes. Concurrent changes under `tools/blender/lods/` and arrow work are unrelated and were left intact. Stop here for knight motion review before applying the slash to another unit or requesting engine edits.
