# Archer reload shoulder roll

Written by GPT-6 Astra.

2026-09-24. The v7 elbow-plane correction was rejected after video review, with `resources/archer_anim_feedback4.png` through `archer_anim_feedback6.png`. The low quiver reach was acceptable, but the forearm still folded behind the upper arm near the head. Passing hinge checks did not make the overall shoulder motion acceptable.

The v8 source separates upper-arm swing, signed shoulder roll and elbow flexion. It interpolates those components between the released pose, quiver grip, an elbow-down extraction pose and the existing high extraction pose. The held arrow tilts backward by up to 24 degrees during the early extraction to clear the forearm. Quiver geometry is unchanged; its orientation is deferred to another thread.

The complete preview is `assets_dev/archer/reload_v8/archer_reload.mp4`, with 265 frames at 30 fps and side and oblique views. Both views use the same final candidate. The user asked to see this existing candidate before any schematic or further revision, so the proposed motion sketch was not built.

The v8 numerical check passes 3,003 sampled poses. Minimum drawing-hand clearance from the torso envelope is 4.306 mm; forearm clearance is 33.191 mm. Approved raise and release tables remain identical. Later reload vertices differ from the v7 baseline by at most 1.333e-15 m. These measurements do not establish human plausibility. The reused L0 mesh remains 2,987 triangles and 1.8 m tall.

After reviewing v8, the user requested removal of the redundant arm raising around video time 00:02 while retaining the roll. This is a refinement request, not approval of the complete reload. The next revision is being prepared in `assets_dev/archer/reload_v9/`, which contains a snapshot of the v8 source and tables. No engine or installed asset changes were made.

`resources/archer_anim_feedback7.png` identifies the above-head pose as the redundant lift. V9 removes the high extraction key at reload time 2.30 seconds. From the elbow-down key at 1.80 seconds, the shoulder swing, roll and elbow flexion interpolate directly to the existing outside-transfer pose at 2.65 seconds. The hand no longer detours above the head. The motion duration remains 5.8 seconds.

V9 passes the motion check with the accepted raise/release baseline and the v8 reload baseline. Reload samples from row 117 (2.65078125 seconds) retain equivalent rotations and identical scalar and arrow channels. Maximum later posed-vertex difference is 1.333e-15 m. Minimum held-arrow clearance against the checked body-part envelopes is 7.845 mm, and maximum free-arrow grip error is 0.898 mm. The reviewed mesh, bow motion, pickup timing and later nock/draw sequence are preserved. The current review target is `assets_dev/archer/reload_v9/archer_reload.mp4`; it requires user playback review before proceeding to another animation.

The v9 side and oblique renders both contain all 265 frames. Saved-scene playback geometry matches the expected samples within 7.153e-7 m across all frames. Inspected sequential frames around the edited section; the complete MP4 is the user review artifact. No in-game playback was performed.

V9 was rejected because the remaining outside-transfer waypoint still raised the hand before carrying it forward. V10 removes that waypoint too. From reload time 1.80 to 3.05 seconds, the grip moves directly from torso-local (-0.500, 1.570, 0.020) to (-0.600, 1.550, 0.300): forward 280 mm and downward 20 mm. The shoulder roll guides the elbow plane, while the grip follows that direct path so blended joint angles cannot produce another upward arc. The arrow tilts outward up to 25 degrees while passing the forearm. All 3,003 motion samples pass, including unchanged raise/release and later-reload comparison from 3.08 seconds. Current review is `assets_dev/archer/reload_v10/archer_reload.mp4`, awaiting user review.
