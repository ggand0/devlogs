# Archer held pose and raise

Written by GPT-6 Astra.

2026-09-23. Gota rejected the original draw motion: the right arm rotated implausibly, the hand crossed the body, and the movement looked like raising an arm rather than drawing. Corrected my mistaken approval record in devlog 0098, the old handoff and the message to Claude.

Following `resources/archer/braveheart_archer_ref.png`, the replacement holds a horizontal drawn-bow pose and raises it without a separate draw or release. Both arm chains use a shared elbow hinge axis; the wrists align with their forearms. The right arm's local rotations remain constant. The torso turns side-on, with separate head/torso parts 18/19 and a cloth overlap at the waist. The held arrow remains visible and nocked throughout.

Gota watched the 45-degree preview and said it looked better, but the waist lean was excessive. He requested a 35-degree limit for the first pass. Reduced the torso lean from 35 to 25 degrees, retaining 10 degrees of bow-arm lift. Re-rendered all 85 frames from both cameras and regenerated the current `archer_raise.mp4`. The earlier video and table are preserved with `_45deg` suffixes.

Current preview: `assets_dev/archer/raise_pose_v4/review.html` and `archer_raise.mp4`. The GLB, `archer.raise.json`, saved Blender scene and validation reports are beside them. Sources are in `tools/blender/archer/`; default builds use `assets_dev/archer/rebuild_raise_v4/`. Source tables are version 2, 65 samples of 30 values, with one raise phase. The old v3 draw/follow contract is superseded.

The mesh has 2,987 L0 triangles. Across 1,001 sampled poses, measured aim is 0 to 35 degrees, draw-arm local rotation change is zero, the elbow hinge-axis error is below 1.5e-15, and draw-hand clearance outside the convex torso envelope is at least 4.306 mm. The wrist/forearm direction error is below 0.000002 degrees. These checks address the reported failure, but do not establish complete anatomical correctness. Gota's video review remains the visual judgment.

This closes the requested first-pass scope at the reduced angle. No runtime integration, final asset acceptance or approval of unrelated stages is implied. No src/ or renderer files, installed assets, commits or pushes were changed. Reload, release, melee, cheers, L1–L3 and full surface/performance acceptance remain separate work. Updated `work/notes/astra-archer-draw-and-release-2026-09-23.md` with the corrected status and current paths.

## Explicit review acceptance and remaining scope

Gota subsequently said "looks good enough for this animation" and asked to update the document and recap the other planned fixes. Recorded explicit acceptance of the current 35-degree raise only in `work/notes/astra-archer-draw-and-release-2026-09-23.md`. The rejected v3 draw/release remains unapproved. The simplified raise replaces the original draw requirement for this first pass.

The original handoff still calls for archer release/follow-through, hip-quiver reload, melee with the bow held clear, and victory cheers for archer, knight, man-at-arms and spearman. Suggested release/follow-through as the next stage, followed by reload, with review at each stage. Engine integration and final asset/performance checks remain separate.

The user requested a fresh Astra session for the remaining animations. Wrote `work/handoffs/HANDOFF-archer-remaining-animations-for-astra-2026-09-23.md`, separate from the Claude message. It preserves the accepted 35-degree baseline, rejected approaches, current sources and table contract, validation limits, remaining stages and engine-edit restrictions. No additional animation changes were made during this handoff.
