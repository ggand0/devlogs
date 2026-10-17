# Man-at-arms left sleeve weights

Written by GPT-6. 2026-10-17.

## Request and diagnosis

Gota reports the earlier knight/spearman animation integration issues resolved and integrated. This task follows handoff 128: the man-at-arms' left arm glitches when playing the knight's attack, block and parry animations. The handoff requests a model-only correction, retaining geometry, atlas, levels and skeleton, with review before an asset-repository commit.

The installed main-tree and graphics-tree MAA models match the frozen `man_at_arms/skeleton_v1/man_at_arms.glb`, SHA-256 `7900cc52914dd49ebeef8b88286ff9a598bfafa00f3c7ae459b0a839048825f8`. The model has the expected original 19-joint family hierarchy.

The original `_skel/knight.py` shield-arm classifier selects one connected sleeve component and treats other components as arm only if every vertex stays within 75 mm of its arm polyline. The MAA has separate padded and mail sleeves; the padded sleeve was classified as shield. Matching exported vertices to the original `textured_v3_sword_rig/source_surfaces.json` identifies 98 affected vertices, all labeled `quilt_sleeve`. There are 307 authored left arm/hand vertices in total.

Handoff 128's broad near-axis count reproduces exactly: 33 shield-dominant vertices among 94 near the upper arm, 54 among 108 near the forearm. However, ten of the latter are genuine `heater_straps` vertices. Therefore its request for both raw cylinder counts to reach zero would incorrectly reweight the straps. The corrected counts are 0/94 and 10/108; all actual sleeve vertices have zero shield influence. The part ID was already correct: knight and MAA sleeves both use `arm_shield`, ID 5, alongside the board. Joint weights distinguish sleeve from shield in skeletal playback.

## Implementation

New iteration `assets_dev/man_at_arms/sleeve_weights_v1/`. `build_sleeve.py` reads the frozen skinned source and original surface labels. It matches all export vertices, including UV/normal duplicates, to authored source positions within one micrometre. It checks that the affected component is the padded sleeve on part 5, then applies the existing knight arm rules through `_arm`.

The rules blend chest to upper arm at the shoulder, upper arm to forearm at the elbow and forearm to hand at the wrist. Parent-child pairs with zero or full weights collapse to a single joint. The resulting effective weights match the mirrored right padded sleeve exactly. All 98 corrections are confined to L0 JOINTS_0 and WEIGHTS_0 records. The build fails if the source hash changes, rather than guessing at an updated model.

Output `man_at_arms.glb`, SHA-256 `e953cebe6e32fb69f82627c3a43fd87c39c603369c497a008aafea458a887150`.

This is a correction to model skin classification, not a clip workaround or extra joint. No shared tool, existing iteration, engine code or installed asset was edited. The old skeleton build alone still reproduces its original misclassification; the new build is the reproducible correction stage. Its source labels and shared helpers are existing dependencies.

## Validation

`check_sleeve.py` reads the saved output again and establishes:

- All JSON metadata and all binary bytes outside the 98 affected weight records are identical. This includes joint nodes, hierarchy, inverse bind matrices, positions, normals, indices, atlas, UVs, colors, part IDs and every other skin weight.
- Original 19 joints retained; normalized weights and valid parent-child influence pairs at every level.
- All 307 left arm/hand vertices have zero shield influence. A shield-only 90-degree turn moves the old sleeve by up to 0.440332 m and the corrected sleeve by exactly zero.
- Corrected effective weights match the mirrored right sleeve with maximum error zero.
- All 12 installed knight clips load and produce finite output at saved keys and 100 Hz interpolation. Every unchanged vertex has exactly the original posed position. This includes block, parry, cut, stab, charge, charge_attack, stand, ready and four shuffles.

Maximum positional correction of the formerly misplaced sleeve is 0.352153 m in block, 0.678946 m in parry, 0.301048 m in cut and 0.276900 m in stab. These values measure the removal of the old erroneous movement, not residual deformation error.

Raw GLB inspection passes the retained channel contract. Triangle counts are 2998/674/248/48 at 1.80 m rest height. Source pivots and all part coverage are unchanged. The surface check records pre-existing L0 open edges (body 348, right arm 20, each leg 40, left arm/shield 20) and ten body same-winding edges. L1–L3 have no reported open/nonmanifold/same-winding edges. No topology repair or new full-model acceptance is claimed.

## Review

The before/after sequence plays the installed knight block, parry, cut and stab with identical timing and cameras. The saved GLB is evaluated with the game's CPU quaternion-blend skin. The orbit turns a full revolution in each direction while clip time advances. Textured rendering uses back-face culling. Every Blender launch was preceded by the host-context game check; no game was running.

Local review: http://127.0.0.1:8768/man_at_arms/sleeve_weights_v1/review.html . Full speed, half speed, orbit and a four-clip sheet are retained in the iteration. The installed clips are identified by SHA-256 in `validation.json`; the inspected main game checkout was `409c81dae4a0e5a91f1c4dfc4e6989311b10ba45`.

Visual review is pending. No in-game test, asset replacement, commit or push has been made. Commit this iteration on assets_dev main after Gota approves it. Claude needs only the corrected model; the knight clips remain unchanged.

## Review verification complete

All three videos decode cleanly and complete a playback and loop cycle in headless Firefox. The localhost page returns HTTP 200 and all local review links resolve. The four-clip comparison sheet and ordered motion frames across the full comparison and both orbit directions were visually inspected. The corrected sleeve remains attached to the arm; the displayed old sleeve detaches or follows the board. This is browser playback and ordered-frame inspection, not an in-game test.

The corrected model and all twelve installed clip hashes were rechecked after rendering and are unchanged. `delivery_checksums.json` records the model and four review media artifacts. Devlog 0257 was backed up after writing handoff 130. The iteration remains uncommitted pending Gota's review, as requested in handoff 128.

Approval update: Gota approved the combined sleeve correction and shield-mount fix from `shield_mount_v1`. This sleeve iteration is archived as its required build dependency. Use the combined model and handoff 132 for integration; the sleeve-only output is historical. See devlog 0258 for approval and the deferred right-arm stab refinement.

Private asset commit: `78f7353fd2098ee9589833d54ab78dda7ea9ae81` (`Fix the man-at-arms sleeve and shield clearance`), on `main`. It archives both build stages and nine review media files through Git LFS. The approved model checksum is unchanged. No push or installed game-file replacement was performed.
