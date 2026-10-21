# Archer knife standing and ready

Written by GPT-6.

2026-10-21. Gota rejected both `quiver_v1` and `sword_draw_v1`, put melee ahead of quiver work, and asked for standing and ready poses with the knife before another draw. The new references are `refs/animation/m2tw_model_refs/peasant_archers_melee/stand_a_idle` and `ready`, each with a GIF and seven-view sheet. The rejected folders remain untouched and uncommitted. Their index outcomes and the earlier integration note now record the rejection.

## Reference observations and correction

The previous work used the bow archer's standing reference and a hand-gunner transition, without an archer melee reference. That was insufficient for the knife guard. The new directory uses `MTW2_Fast_Knifeman`; its index explicitly states that the bow is omitted because the game hides it in melee.

Inspected both complete sheets and ordered GIF frames at the start, roughly one-third, two-thirds and end. Standing is upright, with relaxed arms, feet slightly apart and the knife pointing forward and a little upward beside the right thigh. Ready turns the pelvis and chest to the soldier's right, places the left foot forward, turns the rear foot outward, bends both knees and brings the free hand ahead. The knife stays low near the right hip. It is not the raised frontal guard of the rejected draw.

The authored static poses use frame 0 of each reference for numerical comparison. No curves were transferred and no reference sequence was traced. Body and limb targets are authored on our model. The ready pose uses 140 mm pelvis drop, approximately 38 degrees pelvis turn, 63 degrees authored chest yaw with 25 degrees lean, 630 mm ankle stagger and 330 mm ankle separation across. The rear foot turns 76 degrees right and the lead foot 18 degrees right. The smaller chest lean and our different proportions are visible in the comparison sheets.

## New iteration

`assets_dev/archer/melee_poses_v1/` contains the scripts, `archer.melee_stand.json`, `archer.melee_ready.json`, a review GLB, numeric reports, seven full-body views per pose and a 24-second orbit per pose. Each orbit makes one full camera turn in each direction with a smooth reversal. These are static poses: no draw, idle breathing, transition or attack has been authored.

The model starts from the provisionally approved `hood_v4/archer.glb`. Its body, hood, scarf, skin, pivots and 22-joint hierarchy are retained exactly. The original hip quiver remains while quiver work is paused. The review hides bow parts 8, 13, 14, 15, 17 and held-arrow part 16, matching the melee reference's equipment visibility. It does not reuse the rejected back quiver or bow mount.

The original 52-triangle sheathed hilt is removed, leaving its sheath. A 50-triangle held knife replaces it: handle, guard and pommel plus a closed diamond-section blade, 220 mm from guard to point. It follows the existing `hand_r` joint and part 10 for this L0 pose study, without an added joint or new part id. Independent equipment visibility and matching distance meshes remain integration work; this GLB is not a replacement game asset.

The first ready render left the free wrist curled upward. The final pose straightens its hand axis along the forearm and turns the palm inward/downward. No hand mesh or finger joint was added. The knife and forearms remain clear of the body in both poses.

## Measurements

- 2,994 / 680 / 250 / 56 triangles, 1.80 m rest height, 22 joints.
- Every retained L0 vertex attribute, all L1-L3 arrays, skin and pivot/node data match the approved input exactly. Only the old hilt is replaced and an unused atlas patch supplies the blade colour.
- Closed knife: zero open, nonmanifold, inconsistent-winding or degenerate edges/triangles.
- Both single-sample clips pass the shared validator. Both soles are on the ground within numerical precision; maximum measured penetration is below 0.00001 mm. All four levels have finite posed positions and normals and pass the ground check.
- Visible L0 width is 0.769 m standing and 0.673 m ready. Ready's visible forward bound is 0.661 m and rear bound is -0.324 m. The shared validator includes the hidden bow in its extent report, so its larger width is not the melee review's visible width.
- Zero measured crossings for blade/body, blade/right arm, each forearm/body, forearm/forearm and legs below their covered upper attachments, in both static poses. Intentional hand-hilt contact is excluded.
- Twelve upper-thigh crossings already exist at rest, within about 1 mm of the centre line and rest heights 0.941-0.950 m, under the tunic. Both poses retain twelve crossings in that covered attachment. This is recorded separately instead of calling the entire model intersection-free.
- L0 is narrower than 0.9 m in both poses, so equal-facing lateral copies have disjoint X bounds at 0.9, 1.0, 1.05 and 1.4 m centre spacing. This does not test mixed poses, facing changes or attacks.

The reference checker supports the new melee unit, but its `ours()` implementation treats every archer weapon as the bow. `check_reference.py` corrects only that diagnostic locally, taking +Z through `hand_r` for our knife; shared tools are not edited. Standing has 3.0-degree angular RMS, worst 8.0 degrees. Ready has 4.2-degree RMS, worst 7.0 degrees. Both have zero of 17 rows beyond the existing 12-degree / five-percentage-point tolerances. Reference tables and overlays are under ignored `local_reference/`. These numbers do not establish visual quality by themselves.

## Review and limits

Inspected full-body front, rear, both sides, top and oblique/gameplay renders, including the final wrist and the seven-view reference comparison. `review.html` includes both textured orbits, full-resolution cardinal sheets, reference GIFs/sheets and numeric reports. The reference and our renders both use 2.2 m orthographic framing; their body proportions and target positions differ.

The static poses are the review stage. The original long tunic, hip quiver and approved body proportions differ from the reference model. L1-L3 retain their previous equipment. L3 merges its bow into part 0, so the part-id visibility filter cannot hide that bow; no claim of completed melee LODs is made. No game was launched, no shared engine or simulation code changed, and no simulation hash run was needed.

An automatic approval review rejected an attempted script-copy command because it interpreted it as overwriting the rejected quiver build script. The proposed command read that file and targeted the new folder, but it was not executed. A standalone export script was then written directly in `melee_poses_v1`, avoiding any write to the rejected iteration.

## Branch destination and archive

Gota corrected the eventual destination: bring the approved model source into `feat/archer-rework` in `../flanks` later. The source is currently in public commit `5123c44` on `feat/archer-hood-source`; the approved hood iteration is private commit `f95967a`. This task does not merge, cherry-pick or switch either worktree. The new poses remain uncommitted for review.

Approved input SHA-256: `8ff17dd69e0b69d68da5777e9bdfd615de502796ba7fb22f21aa892815c54be3`.

Knife study GLB SHA-256: `b70f8b7c615f3c34062f6e45525ab72c1daa203b6688e1a5c085f27847311954`.

## Completed media checks

Both MP4s contain 576 frames at 24 fps, 800 by 800 pixels, 24.0 seconds each, and decode completely. All local page links resolve. An isolated headless browser played each video through a complete orbit sequence and into the next loop; ordered browser captures were inspected for both camera directions. The first standing capture clipped the lower body because the capture scrolled before the video layout settled. The video itself was correctly framed. The browser check now waits for layout, centres the video again and asserts that its complete rectangle lies inside the viewport before capturing; the repeated check shows the complete figure. This is offline pose/orbit review, not an in-game or animated-idle test.

No commits or pushes were made. The new iteration remains ready for pose review. Devlogs were backed up to `/data/ggando/flanks/devlogs/`.
