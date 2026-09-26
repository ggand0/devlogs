# Archer reload elbow plane

Written by GPT-6 Astra.

2026-09-24. Gota reported an impossible elbow direction in the earlier reload, using `resources/archer/archer_anim_feedback1.png`. Feedback 2 marked the high extraction pose OK, and feedback 3 described the next pose as weird but acceptable. Gota also said the later reload was good. This was a correction request, not approval of the whole v6 reload.

The arm solver kept a fixed outward elbow pole throughout extraction. Its hinge and flexion checks passed, but the upper arm rolled as the hand crossed shoulder height. Those checks did not constrain the bend plane relative to the torso. The marked frame was around video frame 66, 2.20 seconds into the preview and 1.80 seconds into reload.

Added `reload_draw_pole()` in `tools/blender/archer/motion.py`. It blends toward a down-and-out elbow direction during pickup and early extraction, then returns to the existing outward direction as the arm rises. The correction begins at reload time 1.00 seconds, reaches full influence at 1.60 seconds, and ends at 2.25 seconds. The lowered direction in torso axes is proportional to (-0.6, -1.0, 0.1). The hand path, arrow motion, bow motion and timing remain unchanged.

An initial attempt blended downward too early during the sideward reach and could put the forearm inside the torso. Narrowed the transition to pickup/extraction and checked forearm geometry against the torso envelope throughout the clips. The final drawing-forearm clearance is at least 14.027 mm. During reload time 1.60–1.80 seconds, the elbow remains at least 184.672 mm below the shoulder, instead of holding the upper arm outward while the forearm folds back.

Current review: `assets_dev/archer/reload_v7/archer_reload.mp4` and `review.html`. The corrected section is around 1.4–2.65 seconds in the video; the most relevant frames are 62–70. Reload still runs from 0.40 to 6.20 seconds, followed by the next raise. The preview contains 265 frames at 30 fps with oblique and side views. `archer_reload_review.blend` contains the full sequence. This revision awaits the user's video review.

Saved the previous source under `reload_v7/previous_source/` and previous tables as `previous_reload.json`. Kept the v6 review artifacts intact. The version 3 format, 30 joint channels, 257 reload samples and companion arrow table are unchanged. Approved raise and release samples compare exactly against `accepted_raise_release.json`. The arrow table matches the v6 table exactly. The maximum posed-vertex difference in the later reload, checked from the sample before 2.3 seconds through the endpoint, is 1.333e-15 m. Quaternion signs may differ without changing physical rotations.

Extended `check_motion.py` with forearm clearance and early-extraction elbow-height checks. Its optional `--reload-baseline` argument verifies unchanged later poses and arrow motion against the previous export. All 3,003 sampled poses pass the existing joint, string, grip, rigid-edge and boundary checks as well. All 265 saved scene frames match the sampled geometry within 7.153e-7 m. GLB byte inspection passes for the reused 2,987-triangle L0 mesh. No geometry changes were made, and full asset/LOD/runtime acceptance remains separate.

The main render stopped with exit 143 after side frame 210. Finished side frames 211–264 from the saved scene using the local `finish_frames.py`, then encoded the complete video. Inspected sequential frames through the corrected section in both views. No in-game playback was performed, and the measurements do not establish complete anatomical correctness.

Updated the README's reload commands to v7 and the continuation handoff. No `src/`, renderer or installed assets were edited. No staging, commits or pushes were performed. Melee and cheers remain later stages after reload review.
