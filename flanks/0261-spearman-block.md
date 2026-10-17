# Spearman block from defend_stab_mid

Written by GPT-6. 2026-10-17.

The remaining spear defence from handoff 122 is implemented in `assets_dev/spearman/block_v1/` for visual review. Gota asked for `defend_stab_mid`, separate from the already integrated `defend_slashlr` parry. No game assets, engine code or shared authoring tools changed. No asset commit yet; this iteration awaits review.

## Source and motion

Read the handoff, unit spec, workflow 006 and prior knight/spearman authoring scripts. Viewed the armored-sergeant GIF in ordered frames and its sheet. Rendered and inspected the lighter peasant version as a second reference. Reference renders, overlays and diagnostic measurements stay under the ignored `local_reference/` folder.

The clip draws the level spear back as the shield advances, then lifts the shield in recoil. During recovery the spear briefly rises before returning to the existing ready pose. It is independently authored on our skeleton, with four event comparisons against each reference (ready, guard, recoil and return); no source curves are imported.

The handoff proposed copying the knight shield-arm motion. An initial candidate did that, but its guard shield differed by about 44 degrees and its recoil upper arm by about 71 degrees from the spear reference. The delivered candidate therefore authors the torso and shield arm for defend_stab_mid too. It retains the knight block timing and raise/recoil concept, not identical knight channel values. This change was reported during work.

## Implementation

The installed spearman GLB is copied byte-for-byte, retaining its original 19 joints, skin weights, pivots and approved shield mount. Shield and hand remain siblings under forearm_l. The model hash is checked before building. Existing animations remain compatible because the model does not change.

`build_block.py` uses the established hinged-elbow, forearm-pronation and equipment-orientation helpers from spearman/parry_v1 and knight/block_v3. Hand swing is capped at 30 degrees, while the existing equipment joints control the board and spear direction. No new joint or geometry adjustment.

Guard and recoil are independently authored poses. Smooth interpolation connects ready at 0, guard at 0.20, guard hold through 0.28, recoil at 0.40, recoil hold through 0.50, upright-spear recovery at 0.65, and ready at 0.80 seconds. Leg IK restores both installed ready ankle positions and foot rotations at every saved sample. The clip has 17 samples at 20 Hz, loop false, zero travel, guard 0.20, recoil 0.40 and recover 0.50 events. Both endpoints exactly equal the installed ready sample.

## Measurements and limits

The shared clip validator passes: unit quaternion error below 6.6e-8, no airborne stored frames, exact ready endpoints. Raw GLB inspection was run on the copied model; triangles remain 2992/664/250/56.

Equipment clearance checks cover 401 playback times, or 500 Hz, using the saved JSON and game-equivalent skinning. No sampled board/arm, shield/body, shield/spear or spear/body/arm intersections at L0-L2. The spear-gripping fist is intentionally excluded from shaft/arm checks; straps are not certified by the board/arm test. L3 gets bounds and ground checks, not anatomical collision certification.

Maximum width is 1.832572 m at L0. Across levels, reach is at most 1.505616 m forward and 1.141145 m rearward. Between saved samples, ground penetration reaches 3.097 mm and sole drift 4.616 mm. These small interpolation deviations are reported rather than described as perfectly planted playback.

At 1.4 m formation spacing, left/right neighbors are clear across the tested ready, synchronized and half-cycle-offset cases. Spears cross neighbors ahead or behind in some phases. Tighter 1.05/1.0/0.9 m arrangements also produce side intersections. Existing upper-leg/cloth intersections remain and some poses increase the counts: L0 leg/leg 8 at ready versus 10 maximum, cloth/legs 38 versus 47. These are surface-triangle counts, not penetration depths. The unchanged model's base contact geometry is outside this clip's scope.

At guard and recoil the arm, spear, shield and torso comparison rows are within tolerance. Four rows remain outside tolerance at each event, concerning retained foot placement, rear toe yaw and a shin angle. The existing ready endpoints differ more from the reference, with 8/18 armored-sergeant rows outside tolerance. Per-row reasons and overlays are included in the local review. The stance stays narrower and shorter than the reference so neither endpoint nor either foot anchor changes.

## Review package

The page includes synchronized reference comparison, full and half-speed motion, shield close-up, grounded playback, a 24-second orbit (12 seconds each direction), unadapted-knight versus authored-spear comparison, and seven-view sheets at all four detail levels. The before/after uses the same model in both rows; each motion has its own fixed camera scale. Reference comparison uses 3.660 m framing against the source's 3.600 m.

`compatible/spearman.block.json` is the only new runtime file. The model, ready and stand are unchanged compatibility copies. `spearman.knight_block_diagnostic.json` is for review only. `source_hashes.json` and `delivery_checksums.json` record exact inputs and package files.

## Process note

A first reference-render shell command omitted `set -e` after the game check. It reported running game PID 931910 but continued to launch our Blender PID 936734. I terminated only that Blender process while its log said it was waiting for the running game, before it rendered any frames, and reported the mistake. Subsequent render commands use a checked game-list call before every Blender launch and ran after no game was listed. No other process was stopped.

## Review completion

All eight review videos decode without errors and completed a captured playback cycle and loop in headless Firefox. The ordered motion, inside-shield close-up, both orbit directions and all 17 synchronized reference frames were inspected. No visible board/arm clipping was found in those views. The HTTP review page returned 200 on port 8768. Package hashes were verified again, and the copied model is still byte-identical to the installed model. Nine installed skeletal clips load with its joint list; the non-skeletal arm_stab table is listed separately.

Handoff: `work/handoffs/135-spearman-block-for-review-2026-10-17.md`. Review URL: http://127.0.0.1:8768/spearman/block_v1/review.html. Visual review and subsequent private asset commit remain pending.

## Recovery angle revision, block_v2

Written by GPT-6. 2026-10-17.

Gota found the motion mostly good but marked the steep recovery spear in `refs/spearmen/block_review0.png`. The screenshot corresponds to approximately 0.65 seconds. The original hand-authored direction [-0.24, 0.90, 0.36] raises the spear to 64.32 degrees. V2 changes the recovery direction to [-0.38, 0.55, 0.74], reducing the peak to 33.47 degrees. The existing spear helper coordinates forearm pronation and hand orientation, preserving the grip and anatomical swing limit. This is deliberately shallower than the reference recovery.

Only forearm_r and hand_r values differ in the saved samples at 0.55 through 0.75 seconds. All other channels, root positions, samples through 0.50 and the final endpoint are identical. The model remains byte-identical and has its original 19 joints. `check_revision.py` measures both clips across 401 interpolated times and asserts that the revised peak is below 35 degrees. Shared clip validation and the 401-phase equipment checks pass with zero tested equipment crossings at L0-L2. Existing limits and exclusions remain. Formation checks were not repeated for this narrow recovery change; the changed spear path is not certified against neighboring units.

The previous iteration remains intact. V2 renders a new full-speed/half-speed preview, before/after, shield close-up, reference comparison and 24-second orbit in both directions. Review: http://127.0.0.1:8768/spearman/block_v2/review.html. No game assets, rig or shared tools changed. Visual review and commit are pending.

V2 review completion: all six videos decode and complete a captured browser playback cycle and loop. Inspected the ordered recovery frames, shield close-up and both orbit directions. The page returns HTTP 200, and all package hashes match. Handoff 135 now points to v2 as the current candidate, retaining the original v1 delivery as history.

## Approval and commit

Gota approved block_v2 and requested the commit and Claude handoff on 2026-10-17. Private assets main commit `bc7ddc97ebb70c943c898d2dde33dd7b11b64859` (`Add the spearman block animation`) contains 31 explicitly staged files: the approved iteration and its index entries. All six committed media files are Git LFS pointers. No push; no game asset or source file changed. Earlier rejected block_v1 remains uncommitted and intact.

Commit preparation makes the recovery direction a constructor argument with the approved direction as its default. The regression check reconstructs the former authored direction instead of requiring its untracked rejected clip. Both reconstructed motions were checked against their saved samples and match exactly; the approved clip SHA-256 remains `0c06db5e120f43259168f6b022a7c36f9e5e1896fd5402e54448bc6cec382f18`. All package hashes and the installed model hash match. This packaging change does not change the approved animation or review media. Imported source helpers are already tracked; historical rendered frames and reference material remain local dependencies of the comparison compositor.

Current status: animation approved and committed, game integration pending with Claude. Handoff: `work/handoffs/136-approved-spearman-block-for-claude-2026-10-17.md`. The handoff gives the single JSON replacement, exact SHA-256, unchanged model/19-joint contract, timing, measured limits and in-game formation follow-up.
