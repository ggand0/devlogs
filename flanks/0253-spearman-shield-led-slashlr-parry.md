# Spearman shield-led slashlr parry

Written by GPT-6. 2026-10-16.

## Request and status

After approving the knight 19-joint package, Gota requested the spearman parry next, using `defend_slashlr`. This supersedes handoff 122's block-first ordering. Asked about the handoff's spear-shaft parry brief, Gota explicitly chose “Reproduce defend_slashlr’s shield-led motion.” This candidate follows that decision. The spearman block has not been authored.

The knight approval is recorded in devlog 0252 and private asset commit `f3a1279` (`Restore knight defences to the shared 19-joint skeleton`). The original 19 joint names, order, parents, rest origins and effective skin values were reverified before committing. Workflow 006 now requires explicit permission for joint additions; proposals remain allowed. The workflow and devlogs stay uncommitted under the repository rules.

Spearman iteration: `assets_dev/spearman/parry_v1/`. Review URL: http://127.0.0.1:8768/spearman/parry_v1/review.html . Gota approved the displayed motion on 2026-10-16 and requested documentation, a commit and a handoff to Claude. No shipped-asset replacement or engine edit has been made.

## Reference inspection and authoring

Read the armored-sergeant `defend_slashlr_sheet.png` and all ordered GIF frames. The motion draws the level spear back, brings the shield across the front, sweeps the spear wide at waist height, leans through the deflection, then returns. The reference's visible impact is around 0.35 s and recoil around 0.40 s. It is not a spear-shaft deflection.

The corresponding peasants GIF and sheet were missing from the canonical reference directory. The existing shared renderer already supports the clip, so it was run without editing shared tools, writing only into this iteration's ignored local_reference directory. The generated peasant sheet and complete ordered frames were inspected to expose the legs and left arm. Both units use the same M2TW spear skeleton and corresponding clip.

The authored clip uses the exact ready v5 sample at both ends, two main guard/recoil targets, independent easing and leg IK. Key schedule: ready at 0; guard at 0.15 held through 0.23; recoil at 0.40 held through 0.50; return at 0.90. The initial recoil torso was too upright relative to the sheet; its chest lean was corrected before final rendering. No transferred clip or sampled reference rotation curves enter the package.

Both soles keep their ready anchors at saved samples. The elbow chains are solved from upper-arm/forearm directions. The existing right forearm rolls toward the shaft, the wrist swing is capped at 30 degrees, and the existing weapon joint supplies remaining shaft orientation around the modeled grip. The left hand similarly takes axial roll with a capped 30-degree swing; the existing shield joint controls the board face. There are no new joints.

Four event comparisons per reference: our ready/guard/recoil/return at 0, 0.15, 0.40, 0.90 against source frames 0, 3, 8, 18. Armored-sergeant threshold counts are 8/4/4/8 of 18; peasant counts 7/4/4/7 of 17. Every over-tolerance row has its specific explanation. Endpoint differences are the established ready v5 stance, particularly the more forward/down shield forearm. Guard/recoil differences are lower-body adaptations to the existing 0.32 m width, 0.80 m stagger and planted toe directions. The reference has a longer/wider stance. These are disclosed adaptations, not an exact-match claim.

## Shield mount and compatibility

The first candidate exposed shield-board crossings on the existing spearman, including ready itself (L0 10 crossing points, L1 38 in the checker). The original model is `spearman/ready_v3/spearman.glb`, SHA-256 `11f140624b3b7d826d7c3cf8cb8d01e9f52a6a6af656cb0508b9793b55e656e3`, byte-identical to the installed model at inspection.

`build_mount.py` moves the board 75 mm along its rest normal and 20 mm toward the tip. It extends the board anchor rings of both straps while keeping grip-side rings in place, recomputing strap normals. This is the same geometric principle as the knight mount fix, sized for this model. The full original node hierarchy, glTF skin and per-vertex joint/weight arrays are unchanged. All non-shield geometry and normals, UVs, colors and topology are exact. The model retains 19 joints and 2992/664/250/56 triangles, standing height 1.80 m.

The raw GLB and surface reports were inspected. The L0 arm_shield part has 20 pre-existing open edges in its combined arm/hand geometry; the numerical board check is separate from that combined part. No general topology repair is claimed. Flexible strap contact is not certified by the board clearance checks.

`check_compatibility.py` checks all seven installed spearman clips: stand, ready, stab and four shuffles. All saved keys plus 100 Hz interpolation have zero tested shield-board/arm, shield/body and shield/spear crossings on L0–L2 with the corrected mount. Existing clip files are unchanged. Ready, stand and stab are included with the model; shuffle files need no replacement.

## Validation and limits

The shared clip validator passes. Runtime contract: flanks-clip v1, 19 samples at 20 Hz, 0.90 s, loop false, travel zero, guard 0.15, deflect 0.20, recover 0.50. Exact ready endpoints are separately asserted. The deflection sweep spans the guard-to-recoil interval; the reference impact annotation is not silently substituted for the handoff's event metadata.

At 451 times over 0.90 s (500 Hz), all tested L0–L2 pairs clear: board/arms, shield/body, shield/spear, spear/body, spear/left arm and spear/right upper arm/forearm. The right fist is excluded from spear collision because it grips the shaft. L3 has bounds/ground measurements only. Max width 2.467368 m; forward reach 1.505616 m; rearward reach 1.113129 m. Penetration stays under 1.915 mm. Sole drift is under 3.22 mm in the combined dense reports; saved IK anchors are exact but interpolation produces this small drift.

The 31-phase formation checks show lateral overlap at 1.4 m: ready neighbor right side 7/31, left 0; same-phase and half-cycle-offset neighbors 5/31 on each side. Front/rear overlaps also occur. This reference sweep requires room beside and behind the spearman. Existing upper-leg crossings peak at 11/13/8 points for L0/L1/L2, versus ready 8/11/4. Coat/leg maxima are 43/42/18, versus ready 38/42/18. These are intersection counts, not penetration depths or continuous collision proofs.

## Counter compatibility remains unresolved

Handoff 122 assumes an unblended branch at 0.37 s into parry, entering the existing stab at half its 0.333333 s wind-up (0.166667 s). That condition conflicts with reproducing the clarified shield-led reference: its spear is sweeping laterally at the branch, while the stab points forward. The measured immediate spear displacement is 2.192217 m on L0–L2 and 2.192927 m on L3.

The review shows this exact cut, with no player blend or disguised pose interpolation. The counter renderer evaluates parry directly before the branch and stab directly after it. The sampled diagnostic JSON is for inspection only; it is not a runtime action to integrate. A counter-specific thrust entry authored from this parry pose would preserve the shield motion and avoid a blend, but would need its own timing and integration work. Do not ship the current counter cut as visually solved. Gota was told about this conflict during authoring.

## Review and delivery

The page contains synchronized armored-sergeant/ours playback, half speed, all seven views, a full clockwise/counterclockwise orbit (12 s each), ground-grid playback, original/corrected mount comparison, wrist detail, ready/stab playback and L0–L3 sheets. Reference framing is 3.600 m; ours 3.613 m across to leave a spear margin. All own renders use the saved files and the game's CPU skin. Every Blender launch was preceded by the host-context game check.

The package and diagnostic checksum manifest are `assets_dev/spearman/parry_v1/compatible/` and `delivery_checksums.json`. Main motion: spearman.parry.json. Diagnostics: parry_guard, stab_half_windup and parry_counter_diagnostic. Integration handoff: `work/handoffs/124-spearman-shield-led-parry-for-review-2026-10-16.md`. Gota subsequently approved the motion; see the approval record below. The counter entry remains unresolved. The spearman block remains a separate later item.

## Final review verification

All eleven review videos, including the two ignored reference comparisons, decode and complete a playback/loop cycle in headless Firefox. The page links resolve and its localhost URL returns HTTP 200. Ordered browser captures of the synchronized reference comparison, full-body parry, both-direction orbit, compatibility playback and unblended counter diagnostic were inspected, along with wrist close-ups and sheets. This is browser playback and ordered-capture inspection, not an in-game test.

The ready/stab preview initially used per-clip framing, which changed apparent scale at the clip boundary. Both were rerendered at a fixed 5.5 m scale and the movie was re-encoded through the exact final stab sample. Only that changed video was decoded and replayed again. The renderer now stores framing per clip to prevent later clips overwriting earlier framing metadata.

The isolated shield boards at L0/L1/L2/L3 are watertight with consistent winding (116/72/36/8 board triangles); combined L0 arm-part open edges are separate from the board. All eight delivered file hashes still match the manifest. These checks describe the reviewed files, whose hashes were verified again before archiving.

## Approval and archive

Gota approved this iteration with “ok I think it's pretty good” and requested the implementation record, commit and handoff to Claude. The approved motion and model were archived without changing their bytes. The iteration index records approval. All direct external script and source-clip dependencies are already tracked in the private asset repository. Review media use Git LFS; generated model output, frame sequences and local reference material remain on disk only.

The approval covers the displayed parry. It does not resolve the measured counter-entry discontinuity. Spearman block remains unimplemented. The next handoff is `work/handoffs/125-spearman-parry-integration-for-claude-2026-10-16.md`; it specifies the model/clip pair, unchanged compatible clips and diagnostics to exclude from runtime. No push or game integration was performed.

Private asset commit: `0cd9469dbe46bb3210fd3622efe1216f803a3eba` (`Add the spearman shield-led parry`), on `main`. It contains 56 files, including 13 review media files tracked through Git LFS. Staged whitespace checks passed. All approved package hashes matched before staging. Documentation stays uncommitted as required.
