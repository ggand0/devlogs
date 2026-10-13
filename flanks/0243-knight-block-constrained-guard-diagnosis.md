# Knight block rejection and constrained guard diagnosis

Written by Codex (configured model: GPT-6 Astra).

2026-10-13. Gota rejected block_v1 for an impossible elbow bend and poor correspondence to defend_mid_stab, then requested a fix. The v1 folder is preserved byte-for-byte against its package checksums; only its outcome row in the shared assets index changed. Work continues in assets_dev/knight/block_v2.

## Cause and correction so far

V1 chose the shield upper-arm direction independently, then oriented the forearm to aim the board. The rotations did not preserve the elbow hinge. Its comparison already reported 78.4 degrees upper-arm error and 91.4 degrees forearm error against the intended mid-stab guard. The earlier explanation as an existing sleeve crease was insufficient and misleading: animation rotation was the main error.

The static diagnostic uses the same mid-stab guard-up event, frame 4 at 0.20 s, with independently rounded limb targets. Both arm frames are constructed from one rest hinge and one posed bend plane. The elbow-local rotation preserves that hinge axis. Signed flexion is 79.30 degrees on the right and 110.40 degrees on the left; axis errors are below 0.00001 degrees. The sword's roll is chosen jointly with its forearm rather than independently, reducing wrist swing from 71.54 to 40.90 degrees without changing its blade direction. The shield wrist remains neutral.

The diagnostic restores the crouch and weight shift: pelvis drop 0.25 m, shift -0.02 m sideways / +0.19 m forward; pelvis yaw -55 degrees, chest yaw -78 degrees and forward lean 23 degrees. Both ankle positions remain the ready positions; the rear foot turns to -48 degrees. Leg IK uses the guard's knee planes. At this one measured key pose, all 17 body/limb/weapon comparisons are within the established thresholds. The largest leg direction error is 4.2 degrees. The only remaining over-threshold row is shield facing (82.8 degrees).

## Attachment constraint

The model's forearm and shield are oriented incorrectly for a strapped cross-body guard. The rest forearm direction is 35.80 degrees from the board normal, while the chosen reference guard requires 82.09 degrees. A rigid forearm transform preserves this dot product. With the forearm pointing across the chest and its elbow correctly hinged, the existing board normal becomes approximately (-0.7835, 0.5925, -0.1872), facing sideways/back. The intended guard normal is approximately (-0.24, 0.20, 0.95). This is visible in local_reference/guard_attachment_comparison.png.

Fixing this in another independent animated rotation would separate the board from its straps or transfer the excessive bend elsewhere. A model/attachment correction also affects the accepted ready pose and other existing clips, which is outside handoff 115's fixed model and exact-ready contract. An asynchronous question asks Gota to authorize a corrected shield-arm/model variant and affected ready pose in block_v2. That approval is still pending at this checkpoint. No model geometry, shared skeleton, approved ready pose or shipped asset has changed.

## Artifacts and next step

- guard_diagnostic.py writes the static pose and attachment angle measurements.
- check_hinge.py verifies signed elbow flexion, hinge-axis preservation and wrist swing. It can also sample a finished clip between saved frames.
- render_guard.py evaluates the existing CPU skin and renders the static guard in seven views at the reference's 2.2 m orthographic scale.
- compare_guard.py composes the local-only model/reference comparison.
- knight.guard_diagnostic.json is a diagnostic pose, not a finished or deliverable block clip.

The next step requires the pending model-scope answer. If approved, build the attachment correction in this new iteration, establish the reference-aligned guard and ready pose on it, inspect straps and wrist contact, and assess compatibility with existing knight clips before finishing the block's timed motion. Do not force the current mount to face forward by another independent joint turn.

No commits, push, src changes or game launches. Host game checks precede Blender. The v2 render is diagnostic and is not presented as a finished fix.
