# Knight slash anticipation and cut depth

Written by GPT-6 Astra.

2026-09-24. Gota rejected the first diagonal slash as shallow. `resources/knight_slash_review0.png` asks for a brief hold in the raised pose; review1 marks the steeper downward direction; review2 compares the desired lowered, extended arm against the prototype's bent arm at chest height. Gota explicitly clarified that speed was already sufficient. The change must address the trajectory and follow-through rather than speed up the strike.

V2 keeps the active cut's original 0.16-second timing and adds a 0.14-second raised hold from clip time 0.19 to 0.33 seconds. Total duration becomes 0.74 seconds and contact occurs at 0.41 seconds. The clip now has 149 uniform samples, retaining 5 ms spacing. Tangents are zero at the held raised pose and low follow-through, with continuous travel through contact. The preview timeline derives its length from the clip duration, producing 117 frames at 30 fps with normal and half-speed cuts.

The contact grip moves from 1.415 m to 1.169 m above the ground, a 247 mm drop. Its normalized shoulder-relative offset is (0.24, -0.54, 0.68). The follow-through target is (0.45, -0.70, 0.48), placing the grip at 1.090 m with 32.2 degrees elbow flexion instead of the first version's more bent finish. At contact, the sword axis points 42.15 degrees below horizontal instead of about 16.3 degrees. The cut carries down across the front toward the opposite hip.

Unconstrained cubic interpolation initially exceeded the arm's reach between keys. The hand curve now bounds both Bezier control handles inside 98.5% of the measured reach, reducing the shared tangent at each affected key. Convexity keeps the entire curve reachable without clamping the hand at playback or introducing a joint snap.

Current source remains `tools/blender/melee/sword_slash.py`, with the knight preview and check scripts under `tools/blender/knight/`. The earlier source and table are preserved in `assets_dev/knight/diagonal_slash_v2/previous_source/` and `previous_slash.json`. V1 artifacts remain intact. The v2 GLB is an exact copy of the v1 preview mesh, with no new geometry changes: L0 2,836 triangles, L2 250 triangles, height 1.80 m. The unrelated knight LOD work is untouched.

The 241-pose motion check passes. Maximum held-pose drift is 4.441e-16 m, grip error is zero, elbow hinge error is below 1.37e-15, and loop closure is exact. Elbow flexion ranges from 32.26 to 127.61 degrees. No sampled sword intersections with body, shield or legs were found; feet stay fixed. Inspected sequential key frames in the oblique view before the full render. These checks establish geometry and continuity, not visual approval.

Review target: `assets_dev/knight/diagonal_slash_v2/knight_diagonal_slash.mp4`, with `review.html` for speed control and frame stepping. This revision awaits playback review. No engine, renderer, installed asset, staging, commit or push changes were made.

Both rendered views contain all 117 frames. Saved-scene geometry agrees with the expected sampled poses within 1.431e-6 m. GLB byte inspection was repeated for the copied v2 asset; its bytes match the v1 preview mesh exactly.
