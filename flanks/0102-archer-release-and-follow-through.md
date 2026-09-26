# Archer release and follow-through

Written by GPT-6 Astra.

2026-09-23. The user resumed animation work after the continuation handoff. Authored the next stage only: a 0.6-second release and follow-through beginning at the accepted 35-degree raise endpoint. This release is awaiting user review. The 35-degree raise remains accepted and its 65 table rows match the frozen baseline exactly.

Subsequent review: Gota explicitly accepted `assets_dev/archer/release_v5/archer_release.mp4`, saying "release motion looks good," and authorized proceeding with reload. The release is now accepted. Its source and table were frozen in `assets_dev/archer/reload_v6/accepted_release_source/` and `accepted_raise_release.json` before extending the motion.

The string returns to its braced position over 75 ms. The held arrow disappears at the shot event, leaving the game's separate projectile to take over. The drawing hand moves 40 mm outward, 20 mm up and 25 mm backward in the pre-lean shot frame over 220 ms. A first, larger backward motion folded the elbow past 150 degrees and failed validation; the revised path keeps draw-elbow flexion within 143.194–143.681 degrees. The final world-space hand displacement, including the settling torso, is 45.647 mm.

The bow arm holds aim for 180 ms. Over the remainder of follow-through, torso lean decreases from 25 to 23 degrees and bow-arm pitch from 10 to 8 degrees. The side-on stance and shared elbow hinge solver remain. No separate pulling motion was added.

Source changes are confined to `tools/blender/archer/`: the motion table gains `release` and explicit shot metadata; the sampler respects per-clip arrow visibility; the preview and saved-scene check accept `--release`; motion checks cover both phases and the boundary; a separate HTML player replays the release. The row format stays version 2, 30 values per sample, with 65 rows per phase. Tables still use the filename `archer.raise.json`, now containing both phases. The optional `--baseline` check compares raise samples against a supplied previous table without requiring a local review directory in a fresh checkout.

Preview: `assets_dev/archer/release_v5/archer_release.mp4`, with `review.html` for slow playback, frame stepping and a replay-release button. Shot time is 2.20 seconds. The complete preview is 109 frames at 30 fps: horizontal hold, accepted raise, raised hold, release/follow-through and settled hold. Both oblique and side views frame the entire relaxed bow. `archer_release_review.blend` contains the saved sequence. Full frames are in `frames_oblique/` and `frames_side/`. No flying projectile is included in this asset preview; the player states that explicitly.

The preview mesh is a byte-identical copy of the accepted raise mesh, with SHA-256 `cf9c79d95905c7615981970d3916e9bfe6362fe9d51740d8aaa26ded260c0812`. It remains 2,987 L0 triangles and 1.7999999523 m high. GLB byte inspection passes for its existing attributes and pivots. No geometry or bake changes were needed. The accepted source was copied to `release_v5/accepted_raise_source/` before editing, and its table to `accepted_raise.json`. The original `raise_pose_v4/` review artifacts remain intact.

Validation across 2,002 poses reports zero boundary difference apart from the intentional visibility event, zero changes to the accepted raise samples, minimum draw-hand clearance from the convex torso envelope of 4.306 mm, maximum wrist bend 1.479e-6 degrees and maximum elbow hinge-axis error 1.433e-15. Joint insets remain positive: draw shoulder/elbow/wrist 15.734/13.208/8.880 mm, bow shoulder/elbow/wrist 16.433/12.832/8.880 mm. Rigid edge, joint and string errors stay below 1e-15 m. All 109 saved Blender frames agree with authored geometry within 3.323e-6 m. The full sequence was rendered and sequential release frames inspected; the user's playback review remains necessary. This is not an in-game motion check or complete self-collision/surface acceptance.

Reload, archer melee, all four cheers, L1–L3 and runtime integration remain separate stages. No `src/`, renderer or installed assets were edited by this work. `assets/units/knight.glb` changed concurrently and was left alone. No commit, staging or push was performed.

While recording completion, the previous `tmp/messages/` and `work/handoffs/` directories were no longer present. The attempted multi-file documentation patch failed without changing its targets. Their removal was not performed by this work, and their contents were not recreated over concurrent changes. This devlog records the continuation status instead.
