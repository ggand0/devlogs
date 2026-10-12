# Archer ready pose v1

Written by GPT-6.

2026-10-12. Gota requested handoff 103 item 1: the archer ready pose, bow lowered in the left hand, no arrow nocked, width below 0.9 m. Handoff 106 and note 067 establish that the skeleton and references are delivered. This iteration is `assets_dev/archer/ready_v1/`; it stops for visual review before any shot clip.

## Pose and reference

Read the unit asset spec, animation workflow, skeleton tools and handoffs 093/103/106. Inspected the M2TW ready and standing sheets, plus the ready GIF as ordered frames. The reference is peasant_archers:ready:0, a single static comparison. No motion curves or transferred samples were used.

The authored stance lowers the pelvis by 65 mm, turns pelvis and chest together -45 degrees with 8 degrees of forward lean, and keeps the head facing the target. The left foot is forward: ankle targets (0.16, 0.091, 0.31) and (-0.16, 0.091, -0.31) m. Feet face 0 and -70 degrees. Both soles lie on the ground.

The bow stays upright, with its belly facing along the torso. An initial forward-facing bow put the upper string through the sleeve; turning the bow around its vertical axis removed that crossing. An initial separate chest turn and lean exposed the rigid waist seam at L1; moving pelvis and chest together closed that separation. The draw arm was moved slightly outward to clear the quiver on L1. These were refinements before the first review, within ready_v1.

`Pose.stave()` sets the upright carry and `Pose.string(drawn=False)` braces the string. The player must hide arrow part 16 in ready. The review renderer removes its faces only; the skeleton and model remain unchanged. The bow grip is at (0.19714, 0.89211, 0.59831) m. The string halves meet within 3.4e-16 m.

The final reference comparison has 16 of 17 rows within tolerance, angular RMS 13.5 degrees. The only OVER row is the bow axis, 46.2 degrees from the tilted M2TW carry. This is the explicit upright-bow exception from handoff 103 for agreement with far-level animation. The complete table and overlay stay in ignored local_reference/.

## Measurements

| Level | Total triangles | Visible triangles | Width | Height | Forward reach |
|---|---:|---:|---:|---:|---:|
| L0 | 2,987 | 2,927 | 0.787148 m | 1.768105 m | 0.626562 m |
| L1 | 680 | 680 | 0.762867 m | 1.762901 m | 0.615693 m |
| L2 | 250 | 250 | 0.762847 m | 1.763001 m | 0.615693 m |
| L3 | 56 | 56 | 0.716826 m | 1.762844 m | 0.616659 m |

The shared clip validator passes. It is one sample at 20 fps, 83 floats, 20 joints, no events, zero travel. Maximum sole height error on all four levels is below 0.000057 mm, float serialization noise. Bow tip clearance at L0 is 16.105 mm. The L0 width with the hidden arrow included is 1.42511 m; this is not ready's visible width. Source checksums match the delivered model and tools.

The bow clears the body, legs and draw arm at L0–L2. Both forearms clear the body. L0 bow/arm crossings are zero; L1 has six upper-limb/hand crossings already present at rest. Upper-thigh crossings are 0/11/5 for L0/L1/L2 (rest: 12/11/0). The rigid tunic crosses the legs at 36/29/4 edge intersections (rest: 0/40/20). Those counts measure edges through surfaces, not penetration depth or complete containment. The tunic stays on the pelvis because the rig has no spare coat joints. This is exposed as a surface limitation, not passed as clean geometry.

Ready and standing neighbors, same facing, clear left/right/front/back at spacings 0.9, 1.0, 1.05 and 1.4 m on every level. No transition or in-game playback was tested. This clip is not installed into assets/units and src is unchanged.

## Review and archive

The review page contains seven views on each level, a front/side/oblique summary, a 24-second 640-pixel orbit (12 seconds each direction), local M2TW references and reports. All rendering uses the CPU reference skin and backface culling. Fixed sheet scale is 2.1 m; reference sheets are 2.2 m. Reference material is excluded through the iteration's .gitignore.

Review validation and private archive results are recorded below after completion. Note 052 gives the integration contract and remaining surface limitations. The standing clip and all shared tools remain unchanged.

Final review validation passed: 14 page links, local comparison links, 10 linked images, full FFmpeg decode, and a separate headless Firefox playback through the 24-second orbit and its loop. The video has 576 frames at 24 fps, 640 by 640 pixels. Automated playback advanced from frame 12 to 733. I inspected the seven-view sheets on all four levels, the browser page capture and sampled browser orbit captures in both directions; I did not watch a continuous video feed.

Archived as private assets commit `d4cf6741d2ac2af00f18f94ee31e16e3afc9fb6c`, 27 files including the single README row. The staging check excluded local references, frame sequences, logs and build outputs, and verified all eight review images/videos use LFS. No push. Gota review remains pending.

2026-10-12 review: rejected. The tube-shaped legs and legs clipping through the rigid lower tunic are unacceptable. Do not integrate this candidate as an approved ready pose. Lower-body geometry and cloth skin require revision before animation work continues. See devlog 0218 and note 054.
