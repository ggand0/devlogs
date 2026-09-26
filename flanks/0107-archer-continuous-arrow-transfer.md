# Archer continuous arrow transfer

Written by GPT-6 Astra.

Retrospective: [0120: feedback, techniques and findings across the archer rework](0120-archer-animation-review-lessons.md) connects this final refinement to the earlier rejected draw and reload approaches.

2026-09-24. Gota explicitly accepted the v10 reload as looking alright and acceptable. Preserve `assets_dev/archer/reload_v10/archer_reload.mp4` as the accepted fallback. The requested refinement is shown in `resources/archer/archer_anim_feedback8.png` through `archer_anim_feedback10.png`: after extraction, move toward the front immediately instead of first extending and rotating the arm into the sideways pose in feedback 9.

V11 removes the 3.05-second sideways transfer key. From reload time 1.80 to 3.80 seconds, the grip follows one cubic curve from the elbow-down extracted pose to the low bowstring. In torso-local metres, its endpoints are (-0.500, 1.570, 0.020) and (0.16177044, 1.315, 0.26845436); the controls are (-0.440, 1.550, 0.380) and (-0.100, 1.360, 0.380). The path moves inward and downward throughout. It passes in front of the chest without reaching farther out to the right side. A smooth elbow-pole transition connects the extraction bend plane to the established nocking bend plane.

The arrow starts turning forward at 1.80 seconds. It initially turns toward the front of the torso, then toward the bow direction over 2.70–3.70 seconds. Rotating straight toward the shooting axis too early intersected the drawing arm; the two-stage orientation keeps the shaft clear while the hand follows one continuous path. The quiver geometry and pickup remain unchanged.

All 3,003 motion samples pass. Minimum arrow-shaft clearance from the drawing forearm is 11.290 mm; the minimum across all checked body-part envelopes is 7.845 mm. Maximum free-arrow grip error is 0.695 mm. Raise and release tables match their accepted baseline exactly. The nocked drawing motion after 3.83 seconds matches the v10 baseline to 1.333e-15 m in posed vertices. The L0 mesh remains the same 2,987-triangle GLB. Unused earlier hand-waypoint code was removed; its replacement was already controlling the entire reload.

Review target: `assets_dev/archer/reload_v11/archer_reload.mp4`. This refinement needs playback review; acceptance of v10 does not imply acceptance of v11. No engine, renderer or installed asset changes were made. Melee and cheers remain later animation stages.

The complete preview has 265 frames at 30 fps in both views. Saved-scene geometry agrees with the checked motion within 4.769e-7 m across all frames. Inspected sequential frames of the transfer in both views; no in-game playback was performed. The v10 source snapshot is preserved in `reload_v11/previous_source/`, and its tables are `reload_v11/previous_reload.json`.

Final approval, 2026-09-24: Gota said, "Great, I approve this version." V11 is now the accepted reload. Gota explicitly deferred the archer celebration rework because of the time spent on this animation work. Wrote the current Claude return handoff at `work/handoffs/HANDOFF-archer-approved-ranged-animations-for-claude-2026-09-24.md`, linked it from the historical handoffs/status note, and added a message under `tmp/messages/`. Preserved the approved source in `reload_v11/approved_source/`. This closes the ranged-animation authoring stage; it does not claim engine integration, melee/cheer completion or full LOD acceptance.
