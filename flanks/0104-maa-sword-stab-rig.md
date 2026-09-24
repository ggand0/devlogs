# 0104: Give MAA the sword attack rig

Written by GPT-6 Astra.

2026-09-24. Implemented `tmp/notes/astra-maa-sword-stab-2026-09-23.md` in `assets_dev/man_at_arms/textured_v3_sword_rig/`. The existing MAA carried its sword in the rigid arm part, so its stab read as a small overhead pitch. The existing loader/shader can use the knight's bent-arm attack path when a separate weapon part and the grip/elbow markers are present.

Moved sword blade, grip, guard and pommel into part 8. Added the weapon pivot at GLB (-0.390, 1.084, 0.210) and elbow at (-0.333, 1.228, 0.016). The elbow follows the existing shared sergeant geometry. Kept all other pivots and geometry, including the accepted face/coif and shield.

Final L0: 2,998 triangles, 1.80 m, 1,852 source / 3,660 exported vertices. Parts 0/1/2/3/5/8 have 1,466/276/320/320/462/154 triangles. Exact exported positions, normals, indices, UV0, vertex colors and atlas bytes match the preceding face_fit model. Only weapon UV1 assignments and the two rig markers change. The sword's 154 triangles form closed geometry with consistent winding; none of the hand moves into part 8.

Read the current loader and standing sword shader equations, without editing them. Reproduced those equations in an asset-side preview, with 0.3 s windup and 0.6 s follow-through. The render video plays stab then overhead at half speed in two views, 180 frames per view. Inspected the temporal progression through the draw-back, thrust, recovery and overhead. The grip attachment check passes 2,004 poses with maximum numerical discrepancy 2.3e-16 m. All fist vertices follow the forearm fully. The evaluated Blender playback differs from the sampled transforms by at most 2.7 micrometres due to float32 key interpolation. This is an asset-side preview, not a runtime test; Claude handles the in-game swap/check.

Verified every GLB with the independent inspection helper, checked texture payloads, UV1, part coverage, nodes and sword topology. Preserved all 124 recorded prior files. No src/, renderer, Cargo, runtime animation tables or production asset changes, and no commit. Existing other surface boundaries remain outside the requested rig change.

Handoff: `tmp/handoffs/HANDOFF-maa-sword-stab-2026-09-24.md`. Build commands and pivot/part tables are in the new folder README. The interrupted command left source files but had not exported a model; work resumed from that state without overwriting the baseline.
