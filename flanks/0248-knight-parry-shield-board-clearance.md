# Knight parry shield-board clearance

Written by GPT-6.

2026-10-15. Gota says the overall parry motion is great but identifies the left arm breaking through the shield throughout it. The screenshots are refs/knight/parry_review0.png and parry_review1.png. Both were inspected. The task is to fix the clipping while retaining the motion and a single model for all animations.

## Cause

The board is mounted too close to the wrist and armored glove. In shield-local space the original board back is about 36 mm ahead of the wrist at its center and curves back at its edges. The posed glove reaches about 51 mm in the same direction through most of the parry, and the forearm/glove occupies more space during the downward deflection. The independently rotating shield therefore cuts through those surfaces. The previous checks explicitly omitted shield-versus-shield-arm contact, which allowed the visible defect to pass.

The arm rig is not changed to hide this. Parry v1, block v3 and their model packages remain intact. The new iteration is assets_dev/knight/parry_v2.

## Fixed mount

build_mount.py reads block_v3/compatible/knight.glb and edits only the shield assembly in a new model. The board and studs move 90 mm along the shield's rest normal and 30 mm toward its tip. The grip-side rings of both leather straps stay in place; their board-side ends extend to the relocated anchors. Strap normals are recomputed. All four levels use the same mount placement.

The downward placement matters: an earlier deeper position cleared the glove but briefly grazed the sword between saved frames during the raise and counter. Sampling those intervals at 1 ms resolved the placement before the final build. The fixed mount is constant for every action, not a per-frame displacement or an animation-specific model.

The skeleton, joint origins, skin weights, UVs, atlas, part IDs and triangle counts remain unchanged. All non-shield positions and normals remain exact. All 18 animation and diagnostic-pose JSON files are copied byte for byte, including parry, counter, block and the ten existing clips. No timing, elbow, wrist, body, sword or foot channel changes. The corrected board rides the same shield rotations from its new mounted position.

The shared model and complete compatible clip package are in parry_v2/compatible/. delivery_checksums.json records the files. This remains one model for every action. The shipped assets and game source are untouched.

## Checks

check_mount.py compares every animation byte, skin attribute and non-shield vertex against the source package. It adds the missing board-versus-both-arms check, alongside shield/body and shield/weapon checks. Parry is sampled at 451 times over 0.90 s and counter at 476 times over 0.95 s, every 2 ms. Block is sampled at 81 times, every 10 ms. The ten existing clips and diagnostic poses are sampled at 20 Hz. All three intersection pairs are clear on L0–L2. L3 has only simplified silhouettes and receives ground/bounds and geometry checks.

probe_clearance.py checks the sensitive raise, deflection and counter intervals every 1 ms. The final 90 mm depth / 30 mm drop placement has zero sampled arm or weapon crossings there. Sampling is not continuous collision detection.

The GLB keeps 2,768 / 678 / 250 / 52 triangles and a 1.80 m rest height. glb_inspect.py verifies the actual export attributes. inspect_surfaces.py reports zero open, nonmanifold or inconsistent-winding edges for the complete shield-arm part on L0–L2. The board itself is also checked as a closed component. Existing body/cloth topology outside the shield is unchanged.

Across all four levels, dense parry width peaks at 1.418014 m; counter peaks at 1.418021 m. Ground penetration is at most 2.050 mm for parry/counter and 3.746 mm for block. The larger board offset changes the formation envelope, so check_context.py repeats the 31-phase neighbor check at 1.4, 1.05, 1.0 and 0.9 m. At 1.4 m lateral spacing, ready and synchronized neighbors clear, while half-cycle-offset parries intersect at 2 of 31 phases on each side. Front/rear ranks and tighter lateral spacings also intersect. This is a wider formation envelope than v1 and must be considered during in-game evaluation.

The fixed board clears the arm. This does not establish perfect flexible leather contact against the forearm through every shield tilt. Existing cloth/upper-thigh intersections also remain. These limits are separate from the board penetration shown in the screenshots.

## Review and status

The review page is http://127.0.0.1:8767/parry_v2/review.html. It has before/after videos from matching cameras at full and half speed, seven-view sheets, a complete 24-second clockwise/counterclockwise orbit, and ready/block/stab playback on the same revised model. Animation time runs forward during both orbit directions. The previous reference comparison remains linked.

Rendering uses the game's CPU skin with backface culling. The browser check decodes each video and captures a full playback cycle and wrap. Ordered captures are inspected for the shield front, hand/back attachment, camera framing and motion. This is offline verification, not an in-game test.

Gota approves this revision on 2026-10-15 and requests the commit and a completion handoff to Claude. No push or player integration is included. The assets index has a new v2 row; v1 stays on disk unchanged. No shared source or player changes. The full 0.90-second parry and its reference-style endpoint still require the separately documented runtime timing/entry blend.

Final playback verification: all four MP4s decoded and completed a browser loop. The orbit contains 576 frames at 24 Hz. Ordered captures of the complete comparison, orbit and ready/block/stab playback were inspected; the board front stays clear and both feet remain within frame. Both the page and orbit return HTTP 200. Parry, counter and block saved-clip validators report no failures. Original block package checksums remain unchanged.

## Approval and animation implementation

Gota's final verdict: “this one looks good, I'd approve.” Knight block and parry asset authoring are complete. The corrected parry_v2 model is the common model for both, superseding the model path in handoff 116. The block clip itself is the previously approved block_v3 table, unchanged. The parry is the reviewed parry_v1 motion with the corrected shield assembly. Spearman block/parry are not completed by this work.

Block is independently authored against defend_mid_stab: a fast shield raise, the full knocked-wide recoil with torso twist, and return, 17 samples at 20 Hz over 0.80 s. Events are guard 0.20, recoil 0.40 and recover 0.50. The 20-joint rig adds forearm_twist_l at the elbow between forearm_l and hand_l/shield; pronation is distributed along the sleeve, the elbow bends through its hinge, and hand and board orientation remain separate controls.

Parry is independently authored against defend_mid_slashlr: blade across the front, downward deflection, wide follow-through and recovery, 19 samples at 20 Hz over 0.90 s. Events are guard 0.15, deflect 0.20 and recover 0.50. Three full-body pose targets define start, guard and follow-through, with return to start. Body keys occur at 0, 0.15, 0.35, 0.50 and 0.90 s. Chest orientation includes side lean. Explicit sword-arm targets at 0.15, 0.20, 0.25 and 0.40 s preserve the downward blade path without an elbow-plane reversal. Wrist blade alignment takes the shortest rotation from the forearm. Shield pronation is limited to 85 degrees either way; wrist swing is limited to about 35 degrees during shield authoring. The board has its own orientation. Heel targets and leg IK control the planted front heel and stepping rear foot; coat helpers are baked by the shared Pose tool.

The original parry/knight and parry/militia GIFs and sheets were inspected. Four event poses were compared against each unit (start, guard, follow-through, return); all 18 rows pass. Curves were not transferred. Reference-derived comparison reports, GIFs and sheets stay local and uncommitted.

The counter demonstration branches at parry time 0.15 s toward the existing stab's 0.20-second half-windup. The target clock advances during a 0.20-second blend. The left arm retracts over 0.10 s to clear the sword; per-frame foot IK supports the stance change. At preview time 0.35 s the pose equals the existing stab's 0.40-second contact pose. The preview uses 58 samples at 60 Hz; it is not a replacement stab or a required runtime animation asset. Arbitrary runtime branch times have not been tested here.

Both block and parry have exact internal start/end equality, but their endpoints are their reference-style starts, not canonical ready. They carry zero travel and loop=false. Handoff 115's 0.90-second held block, 0.70-second held parry, exact ready endpoints and release event are not the approved full-motion contract. Integration must resolve timer mapping and entry/exit blending explicitly. The new handoff supplies those differences, the runtime clip list, diagnostic exclusions, hashes and validation evidence.

The approval commit archives parry_v2, its source motion dependencies from parry_v1, and the iteration index. It does not archive the superseded v1 review package or rejected block v1/v2 iterations. Generated GLBs, frame sequences and atlases remain local; review media use Git LFS. The block implementation is already archived in assets commit c069a14. Commit identity and handoff path are recorded below after commit.

## Completion archive

Committed as 97843937e2c6f5dc8e58efaf88d14756b4a18ad8 on assets_dev main: `Add the knight parry with a corrected shield mount`. The commit contains 53 explicit files, including five LFS review-media pointers, the final compatible clip package and the required original motion sources. No raw reference material, generated GLB, atlas, frame sequence, rejected review package or unrelated work was staged. Delivery hashes, existing validation reports, archive dependencies and local review links were verified before the commit. No animation or model change occurred during this documentation/commit step.

Completion handoff: work/handoffs/118-knight-block-and-parry-complete-for-claude-2026-10-15.md. It marks both knight animation assets done and approved, identifies the final common model, lists all runtime files and SHA-256 values, separates diagnostic/counter files, and records the remaining player timing, blending and in-game checks. No push was performed. Devlogs remain outside git and are backed up to /data/ggando/flanks/devlogs/.
