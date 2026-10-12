# Knight charge v2: upright torso

Written by GPT-6 Astra.

Gota found charge v1 bent awkwardly at the waist, pointing to `refs/knight/charge_review0.png` and `charge_review1.png`. Both screenshots were inspected. The cause was authoring 15 degrees of pitch at the chest while preserving an upright neutral pelvis and legs. Matching the reference's chest world lean did not reproduce its whole running posture. The reference number alone was insufficient to judge the body line.

Created a separate `assets_dev/knight/charge_v2/`, preserving v1. Removed chest pitch (15 to 0 degrees) and head pitch (8 to 0 degrees), keeping chest yaw -32 degrees. The same global arm directions, blade direction/roll and shield twist solve are reapplied after the body. Their positions follow the changed shoulder locations. No model, skin, shared tool or engine edit. The future player's forward lean must keep a straight torso; do not reintroduce chest-only pitch over an upright pelvis.

The clip remains one sample at 20 Hz, no loop, events or travel, with root/pelvis/six leg rotations exactly equal to standing. `check_charge.py` additionally asserts that the model-space up axes of pelvis, chest and head all equal vertical. Shared validation passes; unit quaternion error 5.11e-8 and helper error 1.02e-7. Both feet stay grounded, zero penetration at all four levels.

L0 bounds: (-0.549171, 0, -0.383768) to (0.640285, 2.172815, 0.503587) m. Width 1.189456 m; triangles unchanged at 2768 / 678 / 250 / 52. Blade/arm, weapon/body, shield/body, weapon/shield and leg/leg intersections remain zero on L0–L2. Coat/leg counts stay 68 / 104 / 72, equal to standing. Static charge/stand/ready neighbors in four directions clear at 1.4, 1.05 and now 1.0 m. A ready neighbor on the left crosses at 0.9 m (20 / 21 / 17 crossings). This does not establish running clearance or penetration depth.

Reused the same numeric correspondence, charge frame 6 against both M2TW units. The upright chest is now an explicit 15.2-degree difference, bringing OVER rows to nine per comparison. Arm/blade directions remain within 2.9 degrees; chest facing differs by 3.0 degrees. Existing neutral-leg and fixed-shield-mount differences remain documented beside the local tables. No additional reference frame was measured. All reference data remain ignored.

Rendered all levels with the same 2.8 m sheet framing as v1, plus the 24-second, 24 fps two-direction orbit. Added a matched four-view comparison with v1 above and v2 below. The saved pose uses CPU reference skinning. Every Blender launch followed a host process check. No game was launched. The review validation and actual inspected browser captures are recorded in the completion entry below.

Review: `assets_dev/knight/charge_v2/review.html`. Integration note: `work/notes/063-knight-charge-v2-upright-torso-2026-10-11.md`.

Clip SHA256: `c99b746b464356b8f1be47e9903bd0047ccc4693998b9faadb11b6e4aeebb16f`. Model and authoring dependency hashes are in `source_checksums.json`.

The earlier automatic approval rejection of individual-file staging remains unresolved. No staging or commit was retried here. Both charge iterations remain on disk. The review proceeds independently; the charge strike still follows pose approval.

## Review completion

All 32 page links and 29 linked images passed verification. The orbit is 640 x 640, 576 frames, 24 seconds at 24 fps; full decoding passed. Headless Firefox played through both directions and a loop with advancing time/frame counters. Inspected all detail-level sheets, the matched comparison and all 54 timed browser captures as an ordered montage. Torso remains straight and the full figure stays in frame. This is a static pose review, not an in-game gait test or a claim of continuous video perception. Model and dependency hashes remained equal after rendering.

## Approval and next item, 2026-10-11

Gota approved charge v2: "I think it looks good. Will incoporate and see how it'd look with leg work. Approved." The accepted clip and review media are unchanged. Its SHA256 still equals the recorded value above.

This approves the upright held pose. Gota will incorporate it and judge it over the running legs. Gait blending, stride extremes and movement-to-strike continuity remain unverified in the player. Keep the torso straight when combining the pose with the gait; do not restore the chest-only bend from v1.

Updated the private iteration table to accepted for game use and note 063 to the approved integration state. The earlier staging restriction remains unresolved, so this documentation update does not claim an archive commit or engine integration.

Next is `knight.charge_attack.json`, a complete charging strike rather than another held pose: start at approved charge v2, reach contact at 0.4 s, then return to canonical ready over 0.6 s. Handoff 104 assigns Claude the `MTW2_Mace_charge_attack` reference renders for both knight and militia and its addition to the numeric checker. At this approval check, the two canonical reference directories contain no charge_attack GIF/sheet and the checker index has no charge_attack entry. Reference preparation therefore remains outstanding before authoring the strike. A full charge run cycle remains deferred unless the existing gait looks wrong in play.

## Consolidated delivery, 2026-10-11

Handoff 108 now covers the accepted charge v2 pose, provisionally approved charge strike v4 and already integrated stab. The charge pose's bytes and approval are unchanged. A new scoped staging attempt after the explicit commit request was rejected by automatic approval review because of the ambiguous individual-file staging rule in AGENTS.md. No archive commit was created; clarification is required before staging.

## Accepted pose committed, 2026-10-12

The accepted charge v2 folder is committed on private assets_dev/main as `fd4cd391c3e06714abc8b731e8a358ef323b889a`. All eight review media files use LFS. Clip bytes remain unchanged. Gota clarified that rejected iterations stay local, so charge v1 remains untracked on disk. Handoff 108 includes this commit and the provisionally accepted strike v4 commit. No public asset or runtime code changed.
