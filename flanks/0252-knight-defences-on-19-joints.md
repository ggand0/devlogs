# Knight block and parry on the shared 19-joint skeleton

Written by GPT-6. 2026-10-16.

## Request and status

Gota reports that the integrated block and parry look good, but the extra forearm joint prevents reuse by the other unit types. Handoff 122 requires the family to stay on 19 joints. This iteration delivers the knight package for visual review first; the spearman block and parry follow as separate deliveries after review. No shipped asset is replaced and no new iteration commit or push is made before that review.

Iteration: `assets_dev/knight/defend_19_v1/`. Review: `http://127.0.0.1:8767/defend_19_v1/review.html`. Before is the approved `parry_v2/compatible` model and clips, not the earlier mount with clipping.

## Rig and skin implementation

The actual original family skeleton has `hand_l` and `shield` as siblings under `forearm_l`. Handoff 122's phrase about both being under `hand_l` is inconsistent with the installed models. The builder follows the actual hierarchy and joint order, verified against installed man-at-arms and spearman. Their rest lengths can differ; no model migration is required for clip joint-list compatibility.

Remove `forearm_twist_l`. At every sample set new hand quaternion to `twist * old_hand` and new shield quaternion to `twist * old_shield`, normalized. Other channels and the root are copied. The removed twist is axial about the elbow-to-wrist line, so the hand and shield origins stay fixed. Saved joint orientations match within floating-point precision; maximum origin difference is under 0.00004 mm.

Use the original 19-joint skin from `knight/skeleton_v1/knight.glb`. Keep every rest vertex, normal, UV, color and triangle from `parry_v2/compatible/knight.glb`. Thus the board remains 90 mm outward and 30 mm toward the tip, with the corrected strap anchors. No new joint, geometry enlargement or animation-specific model is added. Only the sleeve/forearm/wrist skin deformation changes. Across the moving comparisons the L0 left-arm vertex difference reaches 55.866 mm for block and 49.394 mm for parry; it is roll of the surface, not a change to the elbow/wrist joint path. The wrist close-ups show the extra seam compression.

The model still has 2768 / 678 / 250 / 52 triangles and a 1.80 m standing height. The raw GLB and surface reports were inspected. Rest surface topology is unchanged; pre-existing open coat/leg boundaries are not claimed repaired.

## Motion preservation and its limit

Block remains 17 samples at 20 Hz, 0.80 s, with guard 0.20, recoil 0.40 and recover 0.50. Parry remains 19 samples at 20 Hz, 0.90 s, with guard 0.15, deflect 0.20 and recover 0.50. Both are flanks-clip v1, nonlooping, zero travel. The shared clip validator passes both, including the explicit start/end poses.

All saved joint poses match. Between saved samples, nlerp of a quaternion product is not the product of two nlerps. The dense comparison measures block deviation of 0.03291 degrees and about 0.209 mm at the shield. Parry differs by about 5.02 degrees and 28 mm near 0.225 s; a second deviation of about 4.22 degrees occurs near 0.325 s. The affected intervals are 0.20–0.25 and 0.30–0.35 s. Exact continuous equivalence is not claimed with the fixed 19 samples at 20 Hz. The review renders these intermediate poses at 120 Hz, with full and half-speed videos, so the difference is visible rather than hidden by a keyframe-only movie.

The ten legacy files are byte copies of `assets/units/man_at_arms.<clip>.json` at game revision `7f69eaae66fb902af2bffa85fd4aa389f4acce64`, renamed to knight: stand, ready, four shuffles, charge, charge_attack, cut and stab. All ten produce the same posed vertices as the approved 20-joint package at saved and interpolated times, with maximum numerical difference below 6e-16 m. Their original 19-joint quaternion data is unchanged.

The six ancillary pose/demo files are also converted. `parry_counter` is the old authored demonstration, not a claim about the current unblended runtime counter transition. The runtime consumes block and parry; the other diagnostic poses remain for review and validation.

## Validation and review

`check_clearance.py` tests shield-board versus both arms, full shield versus torso/legs, and shield versus sword at L0–L2. Block, parry and the counter demonstration use 500 Hz sampling; other clips use 20 Hz. All eighteen files pass these crossing checks. L3 has bounds and ground checks only. These are sampled intersections, not a continuous collision proof or a certification of flexible strap contact.

Dense maxima across L0–L3: block width 1.2127394 m and ground -3.8844 mm; parry width 1.4180132 m and ground -2.0499 mm. The foot/leg/body channels are unchanged. The denser block sampling finds a slightly lower intermediate sole than the older 100 Hz report.

Existing coat/leg intersections and neighbor overlap remain documented in `context_checks.json`. At 1.4 m lateral spacing, ready and same-phase neighbors clear; half-cycle-offset parries overlap in 2/31 phases per side. Front/rear and tighter spacing also overlap. This work does not change spacing or the sim.

The review has matching before/after full-body playback at 120 rendered poses per second, half-speed versions, wrist close-ups, small-size playback, seven-view sheets, and twelve-second orbits with a full turn each way. The motion clock continues forward through camera reversal. All renders use the game's CPU skin deformation on the saved model and clips. Game-process checks precede each Blender run.

## Files and next step

The rebuild and review scripts stay inside this iteration. The main graphics worktree's hill-map changes are left alone. The private asset repository stays on main. Handoff: `work/handoffs/123-knight-19-joint-defences-for-review-2026-10-16.md`. Checksum manifest: `assets_dev/knight/defend_19_v1/delivery_checksums.json`.

Gota reviews the wrist deformation and the two short parry interpolation differences. After approval, commit this iteration and its necessary source dependencies in the private asset repository, with no push. Claude integrates the 19-joint model and clips. Then implement the spearman block from armored-sergeants `defend_stab_mid`, followed by the spear-shaft parry; keep the unblended counter branch at 0.37 s in mind for that later clip.

## Review playback verification

All twelve MP4 files decode cleanly and complete a playback-and-loop cycle in a separate headless Firefox instance. The page links resolve, and the local HTTP review URL returns 200. Ordered browser captures of the block orbit, parry orbit and moving before/after comparisons were inspected, along with the wrist close-up renders. The overall shield path remains visually consistent in those comparisons; the localized wrist compression and quantified intermediate shield differences remain explicit review points. All nineteen delivered file checksums match the manifest. The devlog backup was refreshed.

## Approval and original-rig verification

Gota approved this iteration on 2026-10-16, conditional on it using the original 19 joints, and requested its commit. Rechecking the delivered model against `knight/skeleton_v1/knight.glb` confirms exactly the original 19 joint names, order, parents, rest origins and effective per-vertex joint pairs and blend weights. The regenerated first glTF weight differs by at most 5.96e-8 from float32 rounding; the game's effective skin values are exactly identical. All package checksums remain unchanged from review. The condition is satisfied.

At Gota's request, workflow 006 now requires explicit permission before adding joints. A clipping or animation fix is not implicit authorization; proposals remain welcome, with the reason and family-wide migration impact stated first. Internal docs and devlogs remain uncommitted.

Gota requested spearman parry next using `defend_slashlr`, superseding the block-first ordering of handoff 122. He explicitly clarified that the shield-led reference motion should be reproduced, superseding the handoff's spear-shaft deflection.

Archived in private assets commit `f3a127940889b114bf8d7bd9300794243bc8dfa2` (`Restore knight defences to the shared 19-joint skeleton`). No push. Devlogs and workflow 006 remain local under the repository rules.
