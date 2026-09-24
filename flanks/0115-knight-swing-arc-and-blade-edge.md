# Knight swing arc and blade edge

Written by GPT-6 Astra.

2026-09-24. Gota found the v2 hold too long and the intermediate swing unnatural, with the sword appearing horizontal rather than cutting along its trajectory. `resources/knight_slash_review3.png` identifies v2 frame 22. Beginning and ending poses were described as decent. The active strike speed was already sufficient.

The middle hand path shortened the shoulder-to-grip distance, forcing the elbow to fold more before extending. The independently authored blade axis also retained an arbitrary roll inherited from the arm's IK frame. Thus good endpoints and valid hinge geometry did not produce a convincing swing.

V3 replaces the active cut with joint-space motion between the high and low arm configurations. The shoulder rotates along a quaternion arc while the elbow continuously extends from about 82 to 32 degrees. The hand follows the resulting outward swing rather than taking a shortcut toward the shoulder. The blade axis sweeps a rotational arc with the same phase. Its local X cutting edge follows the transverse velocity of a point along the blade, instead of leaving the blade's broad face arbitrarily oriented relative to the cut. Wind-up and recovery blend into that grip orientation.

The top hold is reduced from 140 to 50 ms (0.19–0.24 seconds). Active cut duration remains 160 ms, ending at 0.40 seconds; the contact marker is 0.36 seconds. Total clip duration is 0.65 seconds, with 131 uniform samples at 5 ms spacing. The preview plays a normal-speed cut then a half-speed cut, with oblique and gameplay views. Current review target: `assets_dev/knight/diagonal_slash_v3/knight_diagonal_slash.mp4` and `review.html`.

Updated the reusable motion module, knight review timing, validation and README. V2 source and tables are preserved under `diagonal_slash_v3/previous_source/` and `previous_slash.json`. The development mesh is copied unchanged from v2: L0 2,836 triangles, L2 250 triangles, 1.80 m tall. No model preparation was rerun against concurrent installed-asset work.

The 241-pose check passes for grip contact, hinge alignment, fixed feet, rigid sword lengths, held-pose stability, loop closure and sampled sword collisions against body, shield and legs. Added checks for continuous elbow extension and downward blade-point travel during the cut. Maximum cutting-edge deviation from the sampled sweep, including torso yaw, is 6.534 degrees. Contact grip height is 1.175 m and blade downward angle is 35.28 degrees. The full clip's elbow flexion range is 32.29–114.55 degrees; the maximum authored grip rotation relative to the forearm is 82.01 degrees, including blade roll. These checks are geometric evidence, not anatomical or visual acceptance.

Inspected sequential frames around the reported middle section. The blade necessarily passes through a level orientation while moving from high to low; the change is to its continuous swing, edge alignment and the arm's extension through that passage. No in-game playback was performed. This revision awaits Gota's video review. No engine, renderer, installed asset, staging, commit or push changes were made.

Both views contain all 108 frames. The saved Blender playback matches expected geometry within 1.669e-06 m across all frames. GLB byte inspection was repeated for the copied review asset.
