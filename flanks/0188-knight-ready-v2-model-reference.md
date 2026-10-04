# Knight ready v2 from the posed model reference

Written by GPT-6.

Ready v2 is a new static review candidate, pending Gota's approval. Open `assets_dev/knight/ready_v2/review.html`. It keeps the previous page format with the original 25-degree orbit embedded and the new 45-degree orbit linked alongside it. Each orbit turns clockwise 360 degrees and then counterclockwise 360 degrees over 24 seconds. Stop at this pose for review; no shuffle or cut authoring and no engine integration follows automatically.

Gota supplied `refs/animation/m2tw_model_refs/knight_stand_ready_vs_ours_v2.png`, made by Claude, and explicitly answered 'reference image is the ground truth' when asked about its low forward sword versus handoff 085's up/back wording. That authorizes the low carry used here. It supersedes the sword-direction wording for this iteration. The other width/reach measurements remain useful and pass below. The reference sheet and its individual side/top renders were inspected directly; no M2TW mesh, texture or animation data was copied into the candidate or committed. The earlier stick reference remains background context; no new stick comparison was requested or delivered.

The pose turns the torso 55 degrees, bringing the shield shoulder forward, and leans it 10 degrees forward. Knee flexion lowers the pelvis by 0.14 m, giving a posed L0 height of 1.665333 m from the unchanged 1.80 m rest model. The front shield foot points 5 degrees outward; the rear weapon foot points 45 degrees outward. Each knee lies in its own foot's forward/up plane. The IK solve preserves thigh and shin lengths and the exact target ankles: shield foot (+0.18,0.091,+0.33) m, weapon foot (-0.18,0.091,-0.26) m. The helmet counterturns to face the opponent while retaining the lean.

The sword and hand now share the same complete rotation frame, preserving their source grip relationship. The forearm and wrist place the blade low at the waist, with a world unit direction approximately (-0.22007,+0.10003,+0.97029). The hand remains connected to the elbow solve; no sword-only translation or detached grip was used. Blade/weapon-arm crossings are zero at L0-L2, and the closest L0 blade/arm surface gap is 0.075736 m. Grip containment and exact hand/weapon frame agreement also pass. The reference's lower blade is a deliberate change from ready v1, not a violation hidden by the camera.

The deeper stance exposed cloth problems in draft renders. The final hauberk and surcoat flare and lift as one skirt, with a common affine transform below 0.65 m so the woven band follows its panels. This is pose deformation, not overall model scaling or a change to rest positions. Wider knee/ankle blending bands prevent the lower-leg fold found during sampled checks. The source hem also had sixteen L0 triangles facing into the cloth, visible as floating strips when their parent panel was back-face culled. `prepare_ready.py` reverses those sixteen faces and negates their thirty-two unshared normals. All other GLB bytes are preserved from standing v2. No vertices move, no triangles are added, and the part IDs, atlas, pivots and all L1-L3 bytes remain unchanged. The shipped asset and standing-v2 candidate are not overwritten.

| Level | Triangles | Width, m | Forward reach, m |
|---|---:|---:|---:|
| L0 | 2768 | 0.865448 | 0.658713 |
| L1 | 678 | 0.855812 | 0.658713 |
| L2 | 250 | 0.851791 | 0.658713 |
| L3 | 52 | 0.744419 | 0.334463 |

L0 exceeds the 0.8 m width target by 0.065448 m while staying below the 0.9 m requirement. The blade reaches 0.652411 m forward; the shield front is 0.658713 m, leaving 0.006302 m between their foremost points. No attempt was made to force the width to 0.8 m by undoing the natural skirt spread. The source M2TW model's measured ready width is 1.014 m; this is an adaptation to the knight's proportions, not an exact geometry copy.

Checks pass at L0-L2 for blade/arm, weapon/body, weapon/shield and shield-board/body crossings. The smallest sampled deformation Jacobian determinant is 0.303306. Both soles stay on the ground within floating-point precision. Arm, thigh and shin lengths are unchanged; knee-to-foot-plane errors are below 1e-15 m. The reopened Blender scene matches CPU deformation within 6.073e-8 m. The raw GLB inspector verifies the required channels, including COLOR_0 VEC4 and TEXCOORD_1. Surface inspection still reports inherited open edges and winding exceptions; their counts match ready v1. This is not a claim that the entire source topology is repaired.

The minimum L0 surface gap in aligned 5-by-5 blocks is 0.488028 m at 1.4 m pitch, 0.153854 m at 1.0 m pitch and 0.061596 m at 0.9 m pitch, with no crossings. The closest pair is front to back. L1-L2 pass too. These results cover identical static poses at aligned slots, not moving, differently facing soldiers. L3 remains the standing candidate's static all-body pose; it does not reproduce the close-view crouch. Its review at three pixels is included.

The table changes to ready format version 2, 49 floats, because each leg now needs three world-frame quaternions and the torso needs yaw plus lean. Do not parse it as v1's 35-float layout. `ready_v2.py` and the JSON deformation header are the CPU reference. The shader integration details are in `work/notes/035-knight-ready-v2-2026-10-04.md`. No src files, shipped pose tables or existing attack/shuffle code were changed. All 42 original ready-v1 manifest entries and 38 standing-v2 entries remain unchanged. Existing attacks still have the same rest positions and pivots; no new engine playback or transition verification is claimed here.

Final review verification: inspected front, side, back, top, the composed review sheet and seven ordered frames around each 360-degree camera path. Both H.264 files are 960 by 960, 24 fps, 576 frames and exactly 24 seconds; both decode completely. Camera metadata confirms 25- and 45-degree elevations and an unchanged saved scene. This is a static-pose review using image sequences and encoding checks, not an engine motion test. All local review-page links resolve.

An independent raw-attribute comparison confirms exactly sixteen changed L0 triangle windings and thirty-two negated normals, with every other attribute identical to the standing base and all L1-L3 attributes/indices identical. During this check, the old `mesh_attributes` helper was found to read unsigned-short indices through a normalized-attribute utility and truncate them to zero. The helper now uses the raw integer-index loader. Re-running the base-to-original triangle comparison with actual indices passes; `source_verification.json` records it. Existing private iteration snapshots are preserved.

Source GLB SHA256: `98356597cdda2b8178c861fe01a6bfe0b5fe34f78c2d13bb3706f129969a5446`. Approval is still pending. No files in src or assets/units changed, and no Rust build, game run or sim hash comparison was needed.

Review iteration saved in private assets main as `4c7d51b` (`Add the knight ready pose v2`). All 55 package checksum entries pass. Review media use Git LFS; GLB, Blender scene and frame sequences stay on disk under the repository ignore rules. Public graphics scripts remain on `feat/knight-ready`, with no public asset replacement or integration commit.

Follow-up: Gota finds v2 better than v1 but asks for less knee bend and a relaxed weapon shoulder, with matching model-reference crops on the review page. Ready v3 is recorded in devlog 0189; v2 is preserved and not approved.
