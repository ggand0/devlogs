# Knight mid-slashlr parry v1

Written by GPT-6.

2026-10-13. Gota provisionally approved block_v3, requested its documentation and commit, and requested the knight parry from defend_mid_slashlr.gif and defend_mid_slashlr_sheet.png. He explicitly requires orbit views from this iteration onward. No spearman animation or game integration is included here.

## Previous block package

Block_v3 is committed in the private assets_dev repository on main as c069a14, `Add the knight block with an articulated shield`. Devlog 0245 records provisional approval and handoff 116 records the complete model/clip package, checksums and integration differences. The raw references, rejected iterations and generated model/build output are not committed. The review media use LFS. No push occurred.

## Parry source and implementation

The exact source is defend_mid_slashlr, not defend_mid_slashrl. Both the equipped knight and swordsmen militia GIFs and sheets were inspected. The militia preview was rendered locally from the matching source clip. Four comparison events are start, frame 3 guard, frame 8 follow-through and frame 18 return. References, derived comparison images and paired videos stay ignored in local_reference/.

The new folder is assets_dev/knight/parry_v1. It imports the established elbow hinge solver from block_v3 but does not edit the block, model or existing animation tables. check_artifacts.py verifies every digest in the committed block delivery manifest. The model SHA-256 is 0b18e92d32135a8015f32a938f31606fd279b48d086045f82948feb5e174f312.

The motion includes the fast blade raise across the front, the shield clearing the sword, downward deflection, torso turn, wide follow-through and recovery. Chest orientation includes sideways lean. Elbows use hinge planes; the left forearm twist and shield joints control pronation and shield facing independently. The sword wrist uses the shortest alignment from the forearm rather than a projected orientation that can flip its roll. Legs use heel targets and IK; the front heel stays fixed while its toe turns, and the rear foot steps during the weight shift.

The playable parry table is 19 samples at 20 Hz, 0.90 s, non-looping, zero travel. Its reference-style start and end are identical, but differ from canonical ready. Guard is 0.15 s, deflect 0.20 s and recover 0.50 s. This preserves the full source action, as with the reviewed block, and differs from handoff 115's 0.70-second held parry. Player integration must explicitly reconcile timing and provide the entry blend. No claim is made that it already fits the existing player.

The counter is a separate demonstration. At parry time 0.15 s, it begins blending toward the unchanged stab at time 0.20 s. The stab clock continues during the 0.20-second blend and reaches its unchanged contact pose at preview time 0.35 s. The left arm retracts over 0.10 s to clear the sword. Per-frame foot IK lifts the rear foot during the stance change. This demonstration is sampled at 60 Hz to represent a per-frame visual blend; the actual parry remains 20 Hz. It does not replace the existing stab or prescribe a second animation asset for runtime.

## Validation

Both tables pass the shared validator without changing its thresholds. Parry validation uses parry_start as the endpoint, not canonical ready. The fastest saved parry turn is 1491.91 degrees/s at hand_r; counter is 1316.97 degrees/s at forearm_l. Coat helpers and quaternion lengths pass.

All 18 pose comparison rows pass for all four events against both reference units. Angular RMS is 2.1–3.2 degrees for the equipped knight and 2.8–3.8 for militia. This establishes selected pose measurements, not exact visual equivalence or transferred animation curves.

Dense checks use 200 Hz for joints/contact and 100 Hz for ground, bounds and equipment intersections. Both clips have zero sampled blade/arm, weapon/body, shield/body and shield/weapon crossings on L0–L2. Ground penetration remains below 1.975 mm on all four levels. Parry front-heel drift is 4.525 mm; the toe swings around the heel, with maximum sole-point displacement about 0.21 m. It would be incorrect to call the entire sole planted. Counter front-heel displacement is about 37 mm as it changes into the existing stab stance.

Parry elbow plane error is below 0.000009 degrees. Left elbow flexion spans 64.7–95.7 degrees, right 6.0–83.2. Wrist swing reaches 35.1 degrees left and 47.3 right. The counter enters the existing stab's arm rotations; its larger elbow-plane metric includes the existing axial forearm rotation and is not a new hinge-plane guarantee.

Peak width across levels is 1.3504 m. At 31 L0 phases, lateral neighbors clear at 1.4 m for ready, same-phase and half-cycle-offset states. Front/rear ranks can intersect at 1.4 m; 1.05/1.0/0.9 m lateral spacings also have crossings. The wide follow-through needs an open lane.

Coat and upper-thigh intersections remain and are explicitly reported. L0 thigh hits occupy a 29 by 14 by 22 mm region at 0.776–0.790 m height; these are bounds of intersection locations, not penetration depth. L1 and L2 have broader upper-thigh crossings. Coat/leg maximum crossing counts are 119/138/74 versus ready's 73/122/64 for L0/L1/L2. The model's known grip/strap contact limitation remains. Equipment clearance does not establish correct hand or strap surface contact.

## Review and status

The page is http://127.0.0.1:8767/parry_v1/review.html. It includes synchronized reference/new full and half speed, seven-view motion sheets, ground-grid playback, a 24-second orbit (12 seconds each way, smooth reversal), the counter at full and half speed and the guard/half-windup poses side by side. Animation time continues forward through the camera reversal. Video sizing keeps the full orbit inside the viewport.

All videos are decoded with ffmpeg and played in headless Firefox through a full cycle and wrap. Timed browser captures and ordered reference/new frames are the visual inspection method, not direct real-time human viewing. Source motion, counter sequence and orbit captures were inspected. This is offline CPU skinning, not an in-game test. The server returns HTTP 200 for both block and parry pages, comparison video and orbit.

Parry v1 is awaiting Gota's visual review and is not committed. Its row is in assets_dev/README.md. No game source, shipped assets or sim data changed. Only the already approved block package was committed in this turn. Devlogs remain uncommitted and are backed up at session end.

## Final review outcome

On 2026-10-15 Gota confirms the motion is good and requests correction of the shield-arm clipping. Parry_v2 fixes the common shield mount while preserving all five v1 clip/pose files byte for byte. He approves that revision. The motion source and required clip dependencies are archived at their v1 paths with the v2 commit; the superseded v1 review package stays local. See devlog 0248 and handoff 118 for completion and integration details.

The approved motion dependencies and corrected v2 package are committed as 97843937e2c6f5dc8e58efaf88d14756b4a18ad8 on assets_dev main.
