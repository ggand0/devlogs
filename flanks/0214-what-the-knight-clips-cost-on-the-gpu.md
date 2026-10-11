# What the knight clips cost on the GPU

Written by Claude Fable 5.1. 2026-10-11, branch `feat/skeleton-clips` in the `flanks-gfx` tree at 39930ce (clips on by default), against main 9115bce.

## The question

The perf batch of devlog 0207 (200k, west flank view, 2560x1360, 120 s runs, mean of the last 60 s) read main at 167.1 and 168.4 fps, the branch with the clips off at 164.6 and 166.0, and the branch with the clips on at 156.0 and 154.1. Two gaps to explain: the clips themselves (about 0.40 ms a frame, 10 fps here) and the branch with the clips off still 0.09 ms behind main.

## The answer is already in the logs

Every `fps:` line the game logs every 2 s is followed by the time of each GPU pass that frame (`gpu render/<pass>/elapsed_gpu`). No new run was needed. Mean of the last 60 s of each run, in ms:

| Run | unit draw (`main_unit_pass`) | pose pass (`unit_pose`) | build pass (`unit_build`) | sun shadows |
|---|---|---|---|---|
| main | 4.43, 4.48 | 0.32, 0.33 | 0.16, 0.17 | 0.38, 0.38 |
| branch FL_CLIPS=0 | 4.60, 4.60 | 0.27, 0.30 | 0.15, 0.15 | 0.41, 0.40 |
| branch FL_CLIPS=1 | 4.80, 4.78 | 0.53, 0.66 | 0.16, 0.16 | 0.35, 0.34 |

The scene was the same size in every run: about 177,800 drawn, levels about [3047, 11482, 80k, 77k], GPU clock 1829 to 1863 MHz.

## The three unit passes, in plain words

The GPU draws the soldiers in three steps each frame (render_units_gpu.rs):

1. **Build pass.** One thread per soldier: where he is this frame, is he on screen, which level of detail, and his smoothed signals (walk speed, attack progress, and now his steps). It writes one 64 byte record per soldier.
2. **Pose pass.** One thread per drawn soldier: from his record it works out his whole figure once (legs, arms, lean, and now every joint of the skeleton) and writes it as a list of "pose slots" (16 bytes each) into a pose buffer.
3. **Unit draw.** One run of the vertex shader per corner of his mesh (750 corners for an L2 knight, 156 at L3). Each corner reads the pose slots it needs and moves itself into place. This is the big pass: about 120 million corners a frame in this view.

The sun shadows draw the soldiers again from the sun's point of view, with the same vertex shader, at L2 for the near cascade and L3 for the far ones.

## What skinning is, and what the knight did before

A soldier's mesh is a cloud of corners (vertices). Something has to decide where each corner goes when he moves.

- **Before this branch (and still, for the other three kinds): parts.** Every corner carries a part id: torso, left leg, right leg, weapon arm, shield, and so on. The pose pass computes one turn per part (the thigh angle, the knee bend, the shoulder pitch). In the draw, each corner looks up its part and turns rigidly with it about the part's pivot. A knee is a hinge between two rigid pieces. The draw code for this is a chain of about ten `if part == ...` branches, each doing a couple of rotations.
- **This branch, the knight: a skeleton with skinning.** The knight's model carries 19 joints (pelvis, chest, head, each upper arm, forearm, hand, thigh, shin, foot, coats). Each corner carries which joint, or which two joints, it hangs from and a weight for the second one. The pose pass computes every joint's rotation and position in the world (the hierarchy walk: a hand goes where the forearm puts it, the forearm where the upper arm puts it, and so on). In the draw, each corner takes its joint's rotation and position, and a corner near a joint, like skin over an elbow, blends two joints' rotations by its weight so the surface bends instead of cracking. "Skinning" is that last step: moving the mesh (the skin) with the bones. It is what the clips need: Astra's cut and shuffle clips are rotations per joint over time, which only a skeleton can play.

For the knight, 26 to 31% of the vertices on every level blend two joints; none uses three.

## Is it a one-time cost?

No. Nothing here is paid once. Every one of the three passes runs every frame for every drawn soldier, and the skeleton makes two of them more expensive per knight, every frame:

- The pose pass per knight went from 17 slots to 46 slots (6 common ones, then a rotation and a position for each of 20 joints). To fill them it blends the standing and the ready clip for every joint, blends in the step clip or the cut clip when they play, then walks the joint chain, and writes 46 slots instead of 17.
- The draw per knight corner: instead of the part chain, two or three slot reads (the joint's rotation, the second joint's, the joint's position), a blend of two rotations for a blended corner, and two rotations applied (position and normal).

The one-time costs (reading the GLB, uploading 2380 rows of clip data) happen at load and are not in these numbers.

## Where the 0.40 ms of the clips goes (clips on against off)

- **Pose pass +0.31 ms.** Knights are 40% of the regiments (`FL_HEAVY_FRAC` 0.4), so about 71k drawn knights get the skeleton pose each frame. That is the largest share of the cost by far.
- **Unit draw +0.19 ms.** The skinning of each knight corner, about 30 million corners a frame in this view.
- **Sun shadows 0.07 ms cheaper.** The shadows draw L2 and L3 only, and there the skinned corner is cheaper than the part chain with its branches. This also says the skinning itself is not expensive per corner; the +0.19 ms in the draw is for the finer levels too, where the knight has 8,300 corners (L0) and 2,000 (L1).
- **Build pass +0.01 ms.** The `steps` smoother for every knight.

Sum +0.44 ms of GPU passes against +0.40 ms of frame time: the passes account for the gap.

## Where the 0.09 ms of the branch with the clips off goes (against main)

The unit draw is +0.15 ms, with the same code path taken (the knight uses the part chain, as on main) and the same vertex counts: the skin data (joints and weights) welds no extra vertices, the levels are 3780, 1882, 682 and 146 unique vertices with or without it. The pose pass reads 0.04 ms lower, the build pass 0.015 lower and the shadows 0.03 higher; the net GPU change is about +0.13 ms, and the frame shows +0.09.

The scene is 0.24% bigger with the cut on (the cut is a sim change that stays on with the clips off: about 55 hits a tick against 62, 485 more men alive, 430 fewer bodies), worth about 0.01 ms of the draw.

The rest of the +0.15 ms is unmeasured. The hypothesis: the draw shader of every kind now carries both the skinning code and the part chain, chosen by a runtime branch on a uniform (`rig.skel.params.x > 0`). A shader that carries two paths can need more registers than either path alone, which means fewer corners in flight at once on each GPU core, and the whole 120 million corner pass slows a little. The unit draw pipeline is already specialized per bucket (kind and level), so a compile-time switch (a shader def) per bucket would compile each kind's shader with only its own path. That change tests the hypothesis and fixes it in one go.

## Why the pose pass is slow, hypothesis

71k threads doing a few thousand arithmetic operations each is nothing for this GPU; 0.31 ms says the threads mostly wait. `put_skeleton` (unit_pose.wgsl) keeps three private arrays of 20 entries (`turn`, `rot`, `org`) and indexes them with values that are not compile-time constants: the loop bound is the rig's joint count from a uniform, the gait writes `turn[leg.x]` with the joint index from the rig's role table, and the hierarchy walk reads `rot[p]` with the parent index from the rig. GPU compilers put dynamically indexed arrays in local memory (cached, but every access is a memory round trip) instead of registers. Two ways out, both with the same picture on screen: loop over a constant 20 joints with guards so the arrays can live in registers, or drop the arrays and write each joint into the pose buffer as it is finished, reading the parent back from there. Unmeasured; an A/B run decides.

## Candidates, with expected gains

1. `UNIT_SKINNED` shader def per bucket. Up to 0.15 ms if the register hypothesis holds, else about 0. No visual change. Small change.
2. The pose pass without dynamically indexed private arrays. Unknown share of 0.31 ms. No visual change. Medium change.
3. L3 knights drawn rigid, without the skeleton (the pose pass thread would need to know the level). About 0.13 ms of the pose pass and 0.03 ms of the draw in the flank view, where L3 is 44% of the drawn. Visual: no gait or guard at under 3 px tall. Medium change.
4. The pose packed to 16 bytes a joint (snorm16 rotation, f16 origin). 0.03 to 0.05 ms. Later.

## A/B results (2026-10-11)

Static scene, 30 s runs, passes averaged from 10 s on, `tmp/runs/clips-perf/ab.sh` and `ab_summary.py`; each pair interleaved, two rounds. The scene draws about 174,500 soldiers, levels about [2575, 9497, 86656, 86962]. GPU clock 1874 to 1915 MHz. In ms:

| Build | unit draw | pose pass | sun shadows |
|---|---|---|---|
| base 39930ce | 3.60, 3.55 | 0.94, 0.92 | 0.27, 0.25 |
| base, FL_CLIPS=0 | 3.48, 3.49 | 0.27, 0.26 | 0.24, 0.24 |
| candidate 1, `UNIT_SKINNED` per bucket | 3.60, 3.56 | 0.89, 0.91 | 0.25, 0.27 |
| candidate 2, no private arrays | 3.57, 3.56 | 1.36, 1.36 | 0.24, 0.24 |

- The skeleton costs more here than in the fight: the pose pass is 0.63 ms above the clips-off base (0.31 in the fight). In this scene every knight marches, so the gait branch runs for all of them on top of the clip blends.
- Candidate 1 changes nothing measurable: the hypothesis that the draw shader carrying both paths costs registers was wrong. The split stays in the code only if it is wanted for its own sake; it buys no time.
- Candidate 2 is worse by 0.45 ms: reading each joint back from the pose buffer costs more than the private arrays did. The local-memory hypothesis was wrong, or the arrays never left registers. Reverted.
- Candidate 4, one slot per joint (rotation and origin as halves, `pack_joint`), 26 slots a knight instead of 46: the pose pass means read 0.74 and 0.66 against 0.81 and 0.68, which the noise hides, but the samples pair up in time (both runs see the same march), and sample by sample the packed build is 0.15 and 0.14 ms faster at the median (10 of 13 pairs negative in both rounds). The unit draw is unchanged (3.55, 3.62 against 3.56, 3.62) and the triangle count matches (40.61 M). So the pose pass's writes are a real share of its cost: halving them took about a quarter of the skeleton's 0.63 ms here. The fight's share is about half the static scene's, so expect about 0.08 ms there. Precision: halves put a joint within 1 mm and half a milliradian, 0.03 px at L0.
- The pose pass's samples within a run are bimodal (about 0.4 and about 0.9 in the same run, same scene), which is why run means disagree by 0.15 ms. Pair samples in time and take the median of the differences; means of this pass are not an A/B.
- Candidate 4's first build drew no knights: the unpack helpers sat inside the pose pass's `#ifdef UNIT_POSE_WRITE`, the knight's draw pipelines failed to compile (`no definition in scope for identifier: joint_rot`), and the run read 1.21 ms for the unit draw, 18 M triangles against 40 M. A failed pipeline looks like a win in the pass times; the triangle counter (`clipper_invocations`) and a screenshot caught it. The error check now greps every ` ERROR ` line.

## The answer: animation detail by level (2026-10-11)

Gota, after the four candidates: "I need mostly the same perf as main. Don't propose incremental BS fixes. I need the best fundamental solution." The fundamental fact was in the first table and I looked past it for a morning: 92% of the drawn knights in the flank view are at L2 and L3, under 12 px tall, and every one of them got 19 joints posed and skinned each frame for a motion no one can see at that size. The skeleton pose costs about 6x the part pose per soldier (0.63 against 0.11 ms for the same knights in the static scene), so no tuning of the skeleton code brings 70k of them to main's cost. And the pose pass on main has the same shape: it works out every drawn soldier's full pose at every level, a 3 px man getting the same leg solve as one filling the screen; it never showed because the old pose is cheap.

So the clips play at the levels where they can be seen, as the mesh already has levels:

- `SKEL_LODS` = 2 (render_units_gpu.rs): a skinned kind poses by its skeleton and draws skinned at L0 and L1 (12 px and taller, within about 137 m at 1360 rows). At L2 and L3 he poses and draws by his parts, exactly as a kind without a skeleton and as `FL_CLIPS=0` did.
- The build pass writes the soldier's level into the top two bits of his pose entry (`list_entry`); the pose pass hands it to `pose_write`, which poses the part pose always and the skeleton on top at the near levels. The pose pass's `pose_begin` takes `skel`: without it the steps and the cut clip do not apply, so a far knight's gait and blow are main's.
- The part pose is always written because a near knight's shadow is drawn with the L2 mesh, which draws by parts. The joints follow it: `P_SKEL` = 17, 37 slots a knight (far knights write 17).
- The draw pipeline is one per bucket, so `UNIT_SKINNED` goes to a skinned kind's L0 and L1 buckets only; L2 and L3, and every shadow cascade, compile the part chain.
- A knight crossing 137 m mid-cut swaps motions at 12 px tall. Astra's stand and ready poses should sit near the code pose for that reason; handoffs 102 and 103 say so for the spearman and the archer.

The first capture at contact in the flank view: the near knights on the clips, the far ranks by parts, no error lines.

### The split alone changed nothing: warps

Fight measurement of the split as first built (main 9115bce against it, the batch of devlog 0207, `tmp/runs/clips-perf/fight.sh`): main 170 and 170 fps, the split 159 and 161; the pose pass 0.57 ms against main's 0.31, where before the split it read 0.53 and 0.66. The level bits reached the pass (the picture was right, the far levels drew by parts), and the pass cost the same.

The reason is how a GPU runs a thread: 32 at a time in a warp, every thread of the warp through every branch any one of them takes. The pose slots were handed out in soldier order (`list_entry`, one atomic counter per kind), so a warp of 32 knights held near and far ones mixed, and with 8% near, 93% of the knight warps (1 - 0.92^32) had at least one near knight and ran the skeleton for all 32. The split had moved the work from 100% of the knights to 93% of their warps.

The fix: two counters per kind. The near levels take their slots from the front of the kind's region, the far levels from its back (`POSE_FAR_COUNTER`, the region's capacity `pose_cap` in both passes); the pose pass maps its threads onto the front run and then the back run, so a warp is near or far but not both. The capture at contact with it: 169 fps, the pose pass at 0.21 ms (main's reads 0.31 there), the picture the same.

### Fight measurement with the two-ended slots

Main 9115bce against the split with the two-ended slots, the batch of devlog 0207 (`fight.sh`), mean of the last 60 s:

| Build | fps | frame ms | unit draw | pose pass | build pass | sun shadows |
|---|---|---|---|---|---|---|
| main | 170, 169 | 5.87, 5.93 | 4.46, 4.49 | 0.30, 0.31 | 0.17, 0.17 | 0.32, 0.36 |
| clips at L0 and L1 | 161, 160 | 6.20, 6.24 | 4.60, 4.61 | 0.47, 0.47 | 0.20, 0.20 | 0.37, 0.37 |

From 155 before the split to 160; main is 170. What is left, 0.33 ms:

- Pose pass +0.17. Its samples over the run: main 0.22 to 0.32 throughout; the split 0.28 while the near knights march (same as main's 0.28 then) and 0.45 to 0.52 once they fight, with the L0 and L1 counts flat (3.0k and 11.5k). So the cost is the fighting near knights' step and cut blends (19 joints, two clip samples each, divergent per soldier), not the skeleton pose at rest.
- Unit draw +0.13. The skinned corner at L0 and L1: 3.0k x 8304 and 11.5k x 2034 corners, 48M of the frame's 120M, about 3 ps each over the part corner. The near field of the flank view is knights: cascade 0 casts 2442 knights and nothing else.
- Build pass +0.03. The step smoother runs for every knight, far ones too.
- Shadows +0.03 and the battle 0.24% larger: noise and the cut's sim change.

So the clips' cost is now the near knights': 14.5k of 177k drawn, at 28 px and taller for L0 (within 58 m) and 12 to 28 px for L1 (58 to 137 m). L1 holds four of five of them.

### Probes and the mid-air view

Gota: "Probe first, I want to perf more." Three probe builds, wrong pictures by design, one fight run each in the flank view (the pass times of identical runs had agreed within 0.01 ms):

| Build | fps | frame | pose pass | unit draw |
|---|---|---|---|---|
| branch 6780346 | 161 | 6.20 | 0.47 | 4.60 |
| P1: the rest origin read replaced by a constant | 157 | 6.38 | 0.47 | 4.79 |
| P2: no step and cut blends | 163 | 6.13 | 0.40 | 4.61 |
| P12: both | 161 | 6.23 | 0.25 | 4.81 |

- P2 is clean: the step and cut blends are 0.07 ms of the pose pass's 0.17; the base skeleton pose of the near knights is the other 0.10.
- P1 was a bad probe: with every corner turning about the wrong point the knights cover more pixels, and the draw's +0.19 is fragment work, not the read. A probe has to keep the pixels the same. The clean test of the read is the fix itself: the rest origins as the first rows of the clip buffer, a gather instead of a divergent uniform read, the same picture.
- The pass times move between passes in ways the frame time does not (P12's pose pass 0.25 under main's 0.30 while its draw rose): the GPU overlaps the end of one pass with the start of the next, and the timestamps split the overlap one way or the other. The frame time is the ground truth; the pass times say where to look.

The mid-air view (`FL_CAM_X=-400 FL_CAM_DIST=140 FL_CAM_PITCH=0.28`, same yaw, levels [0, 10040, 78865, 87802]): main 165 fps (6.05 ms), the branch 161 (6.20). Not the "about zero" I had told Gota: at 140 m the nearest soldiers sit at the L1 boundary and 10k of them are L1. The arithmetic fits both views: about 9 ns of pose and 3 ps a skinned corner per near knight (flank: 3k L0 x (9 ns + 8304 x 3 ps) + 11.5k L1 x (9 ns + 2034 x 3 ps) + the build pass's 0.03 = 0.30; mid: 10k L1 x 15 ns = 0.15).

Two fixes with the same picture were then built and measured, one run each: the rest origins as the first rows of the clip buffer (a gather instead of the per-corner uniform read), and the step smoother in the build pass run for the near levels only. Flank fight 6.19 ms against 6.20, mid-air 6.18 against 6.20, the unit draw 4.596 against 4.598: nothing. The uniform read was not a cost and the smoother was already cheap. Both reverted; the branch stays at 6780346.

### Where it ends (2026-10-11)

| View | main | branch 6780346 | cost |
|---|---|---|---|
| flank fight, 200k | 170, 169 fps (5.87, 5.93 ms) | 161, 160, 162 (6.20, 6.24, 6.19) | 0.31 ms |
| mid-air, 200k | 165, 165 (6.05, 6.06) | 161, 162 (6.20, 6.18) | 0.15 ms |

Both are the near knights: 14.5k in the flank view, 10k in the mid-air view, about 15 ns each (9 ns of pose, 3 ps a skinned corner). Zoomed out past 137 m the branch is main. What would move it further is fewer near knights drawn: occlusion culling (planned, cuts the draw too) or the clips at L0 only (about 165 in the flank view, the shuffles and the cut gone beyond 58 m), which is Gota's call.

Gota's questions, answered in the thread: occlusion culling cuts this cost directly, since both passes work on drawn soldiers, and makes the 4.5 ms draw smaller besides; the cost is per near soldier, not per kind, so all four kinds skinned cost about the same as the knight alone; 0.3 ms of a 6 ms frame in the worst view is the price of animation on the men one can see, and 165 is the number to aim at.

## How to measure a candidate

Shaders are embedded in the binary (`embedded_asset!`), so each candidate is a crate rebuild plus runs. The pose pass read 0.53 in one clips-on run and 0.66 in the other, identical binaries; that spread is wider than most candidate gains, so a single fight run per candidate cannot resolve them. Use the static scene (`FL_ENEMY_STATIC=1 FL_AI=0`, as `work/scripts/gpu-pass-measure.sh` does, 30 s runs) for the pass A/Bs, where no knight steps or cuts and the numbers repeat, then one fight run for the final frame time. About 3 to 4 minutes per candidate, 1 minute of it on the GPU. The rules of memory `measuring-perf` hold: `FL_WINDOW=2560x1360`, the flank view, main as the baseline, load and clock logged.
