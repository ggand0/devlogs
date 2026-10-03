# Knight ready pose v1

Written by GPT-6.

Ready_v1 is complete as a static review candidate, not approved. It is in `assets_dev/knight/ready_v1/` on the graphics branch `feat/knight-ready`; start with `review.html` or `ready_review.png`. `standing_ready_comparison.png` puts standing v2 above ready v1. Stop here for Gota's visual review before rebasing any shuffles.

Gota approved standing v2 and chose ready as the next stage. Engine integration is deferred until the basic knight set is complete: standing, ready, side shuffles, forward/backward shuffles and one cut. Gota will coordinate Claude's integration separately and may run it alongside later animation authoring. Standing v2 and its build scripts are unchanged. The asset index now records its approval.

The rewritten handoff 085 was read directly. Ready requirements are width under 0.9 m, blade behind the shield's front with nothing levelled forward, and total forward reach below 0.7 m. Width 0.8 m is a target. This replaces the earlier interpretation of the targets as hard limits. The handoff still contains a sentence saying standing approval is pending; Gota's explicit approval and devlog 0184 supersede that stale sentence.

The M2TW ready sheet and `mace_ready_pose.png` show a staggered stance with the shield foot forward, shield ahead, weapon hand drawn back and torso turned. Ready v1 adapts that stance rather than copying animation data. The body turns 30 degrees, while the helmet stays facing the opponent. The shield foot's ankle is at (+0.15, 0.091, +0.22) m; the weapon foot is at (-0.15, 0.091, -0.18) m. Both soles stay level on the ground. The pelvis lowers by 0.045 m through knee flexion, giving a posed helmet height of 1.755 m from an unchanged 1.80 m source model. No scaling, shortening or joint-coordinate changes were made.

The weapon shoulder and elbow place the hand behind the body; the sword has an independent upward-backward orientation. The shield remains one rigid part, rotated at its shoulder. The first draft exposed helmet/belt distortion from a height-weighted torso turn, skirt/leg intersections, a shield-board/torso intersection, and compression in the rear boot's ankle band. The final deformation turns the body coherently, counterturns the helmet through the neck, lets the skirt follow the thighs, moves the shield clear, and spreads ankle flexion through the lower shin while keeping the sole rigid. These are pose-deformation changes, not rebuilt geometry. The future shader needs the documented deformation; applying only the arm rotations would not reproduce this review.

| Level | Triangles | Width, m | Forward reach, m |
|---|---:|---:|---:|
| L0 | 2768 | 0.623417 | 0.527287 |
| L1 | 678 | 0.622095 | 0.527287 |
| L2 | 250 | 0.610155 | 0.527287 |
| L3 | 52 | 0.744419 | 0.334463 |

All four levels meet the dimensional requirements and the 0.8 m width target. At L0 the entire weapon is behind Z = -0.169579 m, while the shield front is at +0.527287 m, leaving 0.696866 m between their foremost points. L3 is the shared standing candidate's static all-body mesh, with no separate blade or leg articulation. Its silhouette differs from the ready pose at close inspection; the review includes its three-pixel render rather than hiding that limitation.

The minimum complete-surface gaps in identical 5-by-5 formations are 0.609778 m at 1.4 m pitch, 0.222509 m at 1.0 m, and 0.129524 m at 0.9 m for L0. The nearest pair is front to back in each case. L1-L2 also pass; the smallest gap across those levels at 0.9 m is 0.121899 m. The full depth is 1.004922 m, so simple front/back bounds overlap in the tight blocks; actual surfaces remain separate because the shield and rear sword occupy different positions. Edge-crossing checks find no inter-unit crossings at any tested spacing.

Grip containment passes at L0-L2. L0 weapon/body and shield-board/body gaps are 0.045716 m and 0.090242 m, with zero edge crossings. Body, sleeve and leg deformation Jacobians remain positive at the sampled vertices and triangle interiors; the smallest across L0-L2 is 0.300640. Both soles and ankle targets match the ground/targets within numerical precision. The reopened Blender scene matches CPU table evaluation within 0.000064 mm. These checks cover this fixed pose, not motion, transitions, varied facing or the full set of possible cloth/self contacts.

The GLB is byte-identical to the standing-v2 candidate. Comparison with the shipped model confirms the same L0-L2 rest positions, joints, UVs, atlas pixels and triangle connectivity, with the standing candidate's previously recorded normal-export precision differences. L3 retains its standing shield adjustment. Raw GLB inspection confirms COLOR_0 VEC4, TEXCOORD_1, part IDs and pivots. Source topology exceptions remain, including L0 open edges and body winding exceptions; no new blanket topology acceptance is claimed. The shipped model, engine, attack tables and shuffle table remain unchanged.

The 35-float ready table and implementation details are recorded in `work/notes/034-knight-ready-v1-review-2026-10-03.md`. Source scripts are under `tools/blender/knight/`; snapshots and checksums are in the iteration. No game was launched and no Rust build was needed. All Blender runs followed the no-running-game check. The candidate is handed over for visual review only; no next animation stage has been started.

Review package saved in the private assets repository on main as `1809100` (`Add the knight ready pose v1`), with PNGs tracked through LFS. The graphics scripts remain on `feat/knight-ready`; no public asset replacement or engine integration was committed. All 38 entries in standing v2’s checksum manifest remain unchanged. Source GLB SHA256: `72beca756b0207188dd566c2e78a1a0ca5051c2d0ade79dbcde18df7df618690`. Ready-table SHA256: `8e8db381f9ea626521223ab2bf3d8f5eedefb48911b8871ceed1921d4e4574b4`.
