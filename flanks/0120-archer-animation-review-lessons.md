# Archer animation rework: feedback, techniques and findings

Written by GPT-6 Astra.

2026-09-25. Retrospective of the September 23–24 authoring and review work. The individual iterations were already documented, but their feedback and reusable lessons were scattered. This is the entry point for that history. It records the reviewed asset animations, not a fresh audit of current engine integration.

## Accepted result and scope

| Stage | Accepted preview | Decision |
|---|---|---|
| Raise | `assets_dev/archer/raise_pose_v4/archer_raise.mp4` | Hold an already drawn pose horizontally, then raise to 35 degrees. No separate draw is needed for this first pass. |
| Release | `assets_dev/archer/release_v5/archer_release.mp4` | Accepted with “release motion looks good”; proceed to reload. |
| Reload | `assets_dev/archer/reload_v11/archer_reload.mp4` | Explicitly approved with “Great, I approve this version.” V10 remains an accepted fallback. |

Archer celebration was explicitly deferred because this work had already taken substantial time. Quiver orientation was deferred to another thread. Do not interpret approval of these clips as approval of celebrations, a quiver redesign or all possible animation transitions.

The approval-time source snapshot is `assets_dev/archer/reload_v11/approved_source/`. Reproducible source is `tools/blender/archer/`. The authoring contract and accepted files are described in `tmp/handoffs/HANDOFF-archer-approved-ranged-animations-for-claude-2026-09-24.md`. Subsequent engine integration is documented separately in [0108](0108-archer-bow-rig-in-game.md); statements about integration being unfinished in older authoring logs describe that earlier stage.

## Feedback and what changed

### The original draw did not read as drawing

References: `resources/archer_anim_feedback0.png` and `resources/braveheart_archer_ref.png`.

The initial right arm bent in implausible directions, its hand entered the torso during the lift, and the action looked like raising the arm rather than drawing a bow. Gota explicitly corrected the mistaken interpretation of “Ok good” as approval: “I didn't mean to approve it yet.” The geometric checks had passed, but the animation was rejected.

The replacement brief was concrete: make the reference drawn-bow pose horizontally first, then raise the bow toward the pictured angle. Gota said this would already read as reloading without separately animating the draw. The accepted solution therefore changed the action itself instead of continuing to polish the rejected hand path.

The first 45-degree version looked better, but the bend was concentrated at the waist. Gota pointed out that a human would use the entire body and requested roughly 35 degrees for this first pass. The revision uses 25 degrees of torso lean plus 10 degrees of bow-arm lift. The drawing arm keeps its local pose. Gota accepted that version as good enough. See [0098](0098-archer-draw-and-release-prototype.md) and [0100](0100-archer-held-pose-and-raise.md).

### Release needed a small follow-through

The accepted release returns the string over 75 ms and moves the drawing hand a short distance beside the jaw over 220 ms. The bow arm holds aim for 180 ms before settling. A larger backward displacement folded the elbow beyond the 150-degree validation limit, so it was reduced before review. The held arrow disappears at the shot; a separate game projectile takes over. Gota accepted v5. See [0102](0102-archer-release-and-follow-through.md).

### Reload needed a continuous purposeful action

| Revision / reference | Feedback or finding | Response and result |
|---|---|---|
| V6, feedback 1–3 | Later reload looked good, but the early elbow bent in an impossible direction. | V7 adjusted the elbow bend plane while preserving the hand/arrow path and later motion. |
| V7, feedback 4–6 | Still bad: the forearm folded behind the upper arm near the head. | V8 separated upper-arm swing, signed shoulder roll and elbow flexion. Passing a hinge check alone had not fixed the shoulder motion. |
| Axis clarification | Gota first said “roll,” then clarified that it might be pitch: the arm should swing up and back toward the quiver, then arc forward over the top. | Treat the described movement as the requirement, not the anatomical axis label. The existing mesh has a vertical hip quiver; this was not permission to move it to the back. |
| Visualization request | Gota first requested visualization, then specifically asked to see the current candidate as an MP4 quickly before changing it further. | Rendered v8 itself rather than making a new schematic or unreviewed replacement. |
| V8, feedback 7 | The extra lift around video 00:02 was redundant; keep the roll and bring the arm forward. | V9 removed the above-head extraction key. A remaining transfer waypoint still caused an unwanted lift, so v9 was also rejected. |
| V10 | Repeated feedback was to stop lifting and “just bring it forward already.” | Removed the remaining outside-transfer waypoint. The hand moved forward 280 mm and down 20 mm from reload time 1.80 to 3.05 s. V10 was accepted as looking alright. |
| V10, feedback 8–10 | The earlier transfer still looked mechanical. The sideways rotation pose in feedback 9 was redundant; move toward feedback 10 immediately after grasping the arrow. | V11 removed the 3.05-second sideways key and used one continuous curve toward the bowstring. Explicitly approved. |

These screenshot references are `resources/archer_anim_feedback1.png` through `archer_anim_feedback10.png`. Detailed records are [0103](0103-archer-hip-quiver-reload.md), [0105](0105-archer-reload-elbow-plane.md), [0106](0106-archer-reload-shoulder-roll.md) and [0107](0107-archer-continuous-arrow-transfer.md).

## Techniques that worked, and their limits

**Constrain the elbow plane as well as the hinge.** A shared hinge axis and reasonable flexion prevent certain joint errors, but they do not establish a believable shoulder rotation. The fixed outward elbow pole rolled the upper arm as the hand crossed shoulder height. A downward pole helped one section but did not solve the entire action. Early extraction ultimately separated upper-arm swing, signed roll and elbow flexion; the forward transfer uses a smooth bend-plane transition. See `solve()`, `early_reload_angles()` and `forward_transfer_angles()` in `motion.py`.

**Control the hand path when its direction is the complaint.** Removing a named key pose is insufficient if interpolated joint angles or another waypoint still make the hand rise. V10 directly constrained the grip path to move forward and slightly downward. V11 removed the remaining sideways excursion. Judge the actual end-effector trajectory through the edited interval, not only the list of keyframes.

**Use one curve for the intended transfer.** V11 carries the hand from the extracted position to the low bowstring over reload time 1.80–3.80 s. In torso-local metres, the cubic endpoints are `(-0.500, 1.570, 0.020)` and `(0.16177044, 1.315, 0.26845436)`; its controls are `(-0.440, 1.550, 0.380)` and `(-0.100, 1.360, 0.380)`. It moves inward and downward while passing in front of the chest. This removes the sequence of separate lift, extension and turn gestures that made the action mechanical.

**Author arrow orientation separately from hand position.** Turning the arrow directly toward the shooting axis too early intersected the drawing arm. V11 starts turning it forward at 1.80 s, then turns toward the bow during 2.70–3.70 s. The hand follows one continuous path while the shaft clears the arm. The free-arrow transform blends to the string-derived transform at nocking, complete at 3.80 s. This needs the companion arrow table, not just arm rotations.

**Use local quaternions and an explicit hierarchy.** The original Euler interpolation produced an intermediate excursion. Exported arm rotations use local XYZW quaternions with shortest-path normalized interpolation and parent-times-child composition. The final format has 30 joint/scalar values per row, 65 samples for raise/release and 257 for reload, plus 8 values per reload arrow sample. Visibility changes at events, rather than fading through interpolation. Runtime playback can use forward kinematics; the authoring solver does not need to run in the game.

**Keep equipment contacts and geometry measurable.** The bow grip follows its hand; two fixed-length string segments determine the nock. Held-arrow visibility is independent of decorative arrows in the quiver. Separate rigid arm parts and joint covers suit this low-poly model, but contact and cover checks still need to accompany the pose work. The accepted mesh is 2,987 L0 triangles and 1.800 m tall. Raise, release and reload reuse that same mesh.

## Review and verification lessons

- Passing hinge, rigid-edge, contact or clearance checks does not establish human plausibility. Both v6 and v7 demonstrate this. Gota's moving-image feedback overruled those limited checks.
- Review the whole sequence from oblique and side views, including intermediate frames. Good endpoints can conceal a bad shoulder roll, hand/body crossing or redundant lift. The authoring tools rendered complete MP4s and inspected ordered frames; do not describe that as an in-game playback test.
- Match video time to clip time. The reload preview begins the clip at video 0.40 s, so video 00:02 is around reload time 1.60 s. Its later holds are preview presentation, not the simulation cadence.
- Preserve accepted sections with source/table snapshots and numerical baseline comparisons. Final v11 leaves the accepted raise/release tables exactly unchanged; nocked drawing after 3.83 s matches v10 within 1.333e-15 m in posed vertices.
- When removing an unwanted gesture, inspect every remaining waypoint and the interpolated path. Repeatedly removing only part of the lift prolonged this review and frustrated the user. Resolve the entire reported interval before returning another candidate.
- Give the user the requested current MP4 quickly when the question is whether the current motion already works. Do not substitute a new sketch or another interpretation for that review artifact.
- Record explicit acceptance and its scope. V10 was accepted as a fallback, v11 was explicitly approved, and celebration/quiver work was deferred. An encouraging interim response is not approval.

Final v11 checks covered 3,003 sampled poses. Minimum torso-envelope hand/forearm clearance was 4.306/33.191 mm; minimum checked arrow-shaft clearance was 7.845 mm. Maximum free-arrow grip error was 0.695 mm. All 265 saved playback frames matched authored positions within 4.769e-7 m. These figures quantify the checked constraints; they are not a complete self-collision, anatomical or final-asset acceptance claim.
