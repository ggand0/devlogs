# Man-at-arms shield clearance

Written by GPT-6. 2026-10-17.

Gota confirms the earlier sleeve glitch appears fixed but reports the same left arm/board clipping previously corrected on the knight. The earlier candidate fixed the misplaced padded sleeve weights but retained the original shield geometry. This follow-up fixes the mount in a separate iteration, `assets_dev/man_at_arms/shield_mount_v1/`, preserving the earlier review files.

## Cause and correction

The old mount crosses the arm even in ready and throughout sampled block/parry motion. The 20 Hz baseline in `clearance_before.json` reproduces board/arm crossings at L0/L1 in all twelve installed knight clips, and also at L2 during block and parry. No shield/body or shield/sword crossings were reported.

The new build reads `sleeve_weights_v1/man_at_arms.glb`, SHA-256 `e953cebe6e32fb69f82627c3a43fd87c39c603369c497a008aafea458a887150`, so it includes the 98 corrected padded sleeve vertices. It uses the knight's mount principle: move the board out along its rest normal and down toward its tip, while extending board-side strap anchors and preserving the grip-side rings. No arm pose, weight or skeleton change is needed for this stage.

The knight's exact 90 mm outward/30 mm down adjustment initially cleared L0 but still intersected the MAA's larger L1/L2 arm meshes in parry at 0.25 s. That result is retained in `clearance_initial_90mm.json`. The final adjustment is 100 mm outward along `[0.4226182617, 0, 0.9063077870]` and 30 mm downward at every detail level. Its output hash is `374cdca0cf711430a2406faf0feb2042c12e44268777f658450e21b0fac4bb55`.

Each of L0–L2 has two straps. Moved vertices total 244/188/104/20 at L0/L1/L2/L3. The moved strap anchor counts are 44/14/17, while 68/34/29 grip-side strap vertices remain fixed. Only strap normals are recomputed. The board, back, rim and fittings retain their shapes; the board moves as a complete solid.

## Validation

`check_mount.py` verifies the same original 19 joints, skin, hierarchy and pivot nodes. All weights, non-shield geometry/normals, atlas, UVs, colors and indices remain exact relative to sleeve_weights_v1. Binary bytes outside the position/normal accessors are exact. Geometry changes are restricted to the shield assembly; strap grip rings remain fixed. The sleeve-weight correction remains intact.

`check_clearance.py --hz 500` passes all twelve installed knight clips at 4,689 times per level, including every saved sample and interpolation at 500 Hz. Board/arm, shield/body/legs and shield/sword crossing counts are zero throughout L0–L2. L3 receives bounds and ground checks only. All isolated shield boards are watertight with consistent winding.

The twelve clips are block, parry, cut, stab, ready, stand, charge, charge_attack and four shuffles. Their file hashes are recorded in the report. The test uses the game's CPU quaternion skin and saved clips. No simulation or animation code changes are needed.

Triangle counts remain 2998/674/248/48 at 1.80 m rest height. L0 maximum widths are 1.218572 m in block and 1.389400 m in parry. Their existing ground penetration is 3.645 mm and 1.991 mm respectively. Surface inspection retains pre-existing L0 body/limb defects; no general topology repair is claimed. Flexible strap contact is not certified by the board clearance check. The result is sampled validation, not continuous collision detection.

## Review and delivery

The new page compares the sleeve-corrected model before and after the mount correction. It includes full-speed block/parry/cut/stab, half speed, an arm close-up and full clockwise/counterclockwise orbits. Model deformation is evaluated from the saved GLB with back-face culling. Before each Blender launch the host game check reported no game running.

Review: http://127.0.0.1:8768/man_at_arms/shield_mount_v1/review.html . Delivery is the iteration's `man_at_arms.glb`. It replaces the previous sleeve-only candidate for eventual game integration. Handoff 131 records the combined model and checksum. The game files, shared tools and old iterations remain unchanged. Gota subsequently approved the combined result; see the approval record below. No push or game integration has been performed.

Final review verification: all four videos decode cleanly and complete a playback/loop cycle in headless Firefox. The review page returns HTTP 200 and its links resolve. The full ordered close-up and orbit sequences were inspected; they show the arm behind the board with the mount anchors extended. This is browser playback and ordered-frame inspection, not an in-game test. The model and all twelve clip hashes were verified unchanged after rendering. `delivery_checksums.json` records the model and five review media files.

## Approval and deferred stab refinement

Gota approved the combined version: “Good it's fixed now. I'd approve this version.” He requested documentation, a commit and a handoff to Claude. The approved model and review media were rechecked against their saved checksums before archiving; every hash matches. Both the sleeve correction and mount iteration are included because the latter builds on the former. External geometry-label and shared-helper dependencies are already tracked. Generated models and frame sequences stay on disk; review media use Git LFS.

Gota also noted that the MAA's right arm looks slightly weird during the stab animation. He considers it good enough for now and may want to refine it later. Record this as a deferred visual animation improvement, not an integration blocker and not a diagnosed skin or player defect. No stab animation change was requested or made in this archive step. The note is also in the iteration README and Claude's approved integration handoff 132.

Private asset commit: `78f7353fd2098ee9589833d54ab78dda7ea9ae81` (`Fix the man-at-arms sleeve and shield clearance`), on `main`. It archives both build stages and nine review media files through Git LFS. The approved model checksum is unchanged. No push or installed game-file replacement was performed.
