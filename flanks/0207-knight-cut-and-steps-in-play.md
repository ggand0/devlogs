# Knight cut and steps in play

Written by Claude Opus 5.5.

Steps 2 and 3 of handoff 095: Astra's accepted cut and four step clips now play in the game behind `FL_CLIPS=1`. Branch `feat/skeleton-clips` in the `flanks-gfx` tree: 8f581d6 (what Gota played first) and b1be7f2 (the cut sped up after that play).

## What Gota settled (2026-10-05)

- The cut is a slow, strong blow beside the fast stab. The stab keeps today's timing and arm motion.
- A charging knight always stabs: a slow cut at a run would carry him into the enemy before it lands.
- Stab or cut stays a 50/50 pick, as today.
- The steps follow M2TW: a man in a fight sidesteps below the speed where he turns to walk; four clips, the closest one, never mixed; when he stops, the foot in the air lands, then he settles into his guard. M2TW's knight (`MTW2_Swordsman`, parent `MTW2_Mace` in `descr_skeleton.txt`) has exactly `shuffle_forward/backward/left/right`, no diagonals. When it switches and how it stops lives in the engine, not the data.

## The cut (sim change, knights only)

- `UnitTypeParams::cut_windup_ticks` 36 and `cut_return_ticks` 42 for the knight: the clip's contact at 1.20 s and its return of 1.40 s, so it plays at the speed it was made. 0 for every other kind, which keep their style 1 swing.
- Style 1 of a kind with a cut is the cut (`is_cut`): wind-up `windup_for(swing)`, the pause after it never shorter than the return.
- `cut_weight()`: the cut lands harder by the ratio of the mean cycles, so both blows deal the same damage over time. Knight: (36 + 48.75) / (12 + 48) = 1.41. The 48.75 is the jittered pause held at 42 or above (a quarter of draws fall under 42). It rides the damage event's `jit`. Test `cut_weight_matches_the_mean_cycles` checks it against a brute-force average.
- `COOLDOWN_JITTER` names the ±25% the sim already drew; the arithmetic is unchanged.
- Fingerprints with knights in a fight change by design. Other kinds draw the same numbers as before.

## The cut on screen

- Both render paths take the cut's wind-up and its return (`follow_s` in `render_units.rs`, `is_cut` and `follow_s` in `unit_build.wgsl`, per kind through `BuildParams::cut_windup`, `cut_return`). Without the flag the old knight plays the old swing at the new speed.
- `put_skeleton`: clip time = wind-up x contact before the blow, contact + follow x (length - contact) after it. Blended in over the first 20% of the wind-up and out over the last 25% of the return; both ends are the ready pose. The guard stays up during a cut. On the move the pelvis, legs and coat keep the gait.
- `put_common` drops the blow's body lean for a cut, since the clip moves the whole body.

## The steps (render only, GPU path with its pose pass)

- `steps()` in `unit_build.wgsl`, state in `Smooth` (`step`, `step_clip`, `step_w`; the buffer stride is now `SMOOTH_BYTES` 52). He steps when his band is above 0.15 (an enemy within watch range), his smoothed speed is under `STEP_FACE_SPEED` (1.05 m/s, now `pub(crate)`), and he moves faster than 0.06 m/s.
- The clip is the closest of the four to his motion relative to his facing. A play advances by ground covered / the clip's travel (0.37 m at main's knight size). He changes clip only when a play ends, which is the ready pose in all four.
- When he stops or speeds into his gait, the step under way runs at the clip's own pace to the next half play (both feet down), then his weight in the steps eases to 0 over `STEP_SETTLE_S` 0.25 s. Going in eases over the same time.
- The step state rides the record's fourth position float for a skinned kind (an archer's bow uses it otherwise): clip + 4 x place + 4096 x weight, both in 1023rds, exact in f32.
- `pose_begin` decodes it and scales the gait by 1 - weight (walk gate, arm swing, hip dip, shoulder turn and shift). `put_skeleton` blends the step clip over the stance before the gait.
- The CPU path has no skeleton, so it plays no steps.

## After the first play (2026-10-05)

Gota: the steps look good so far but need a longer play to judge; the cut at its authored speed is too slow, and it fits a shorter time than I had said. Set to 0.6 s to the blow as an experiment: 18 ticks to the blow and 21 back, the clip at twice its speed throughout (b1be7f2). The return no longer bounds the pause (21 is under the shortest jittered cooldown, 36), so the weight is (18 + 48) / (12 + 48) = 1.10.

## 0.4 s (2026-10-06)

Gota asked whether 0.6 s is realistic for a one-handed diagonal cut. From general knowledge of strike timings (not measured): about 0.4 to 0.5 s from the start of the wind-up to the hit from a guard, the strike itself about 0.2 s. Set to 0.4 s to try (12 ticks to the blow, 14 back, the clip at three times its speed). The cut now has the stab's timing, so its weight is 1.0: it differs from the stab only in look. A slow-motion video of test cuts, counted frame by frame, would settle the number.

## 0.5 s (2026-10-06)

Gota: 0.4 s is too fast in play. Set to 0.5 s: 15 ticks to the blow and 18 back, the clip at 2.4 times its speed, weight (15 + 48) / (12 + 48) = 1.05.

## Back to 0.6 s (2026-10-06)

Gota, after playing all three: 0.4 and 0.5 s clearly look like a sped-up clip; 0.6 s matches this animation. Back to 18 ticks to the blow and 21 back, weight 1.10. A real cut can be faster, but this clip's motion sets the pace: a faster cut needs a clip made for it.

## Checks

- opt-dev build, `cargo clippy --profile opt-dev -- -D warnings` clean, 36 tests pass.
- Not launched by me (Gota asked for a build to play); Gota's play ran 8f581d6 with no shader trouble. b1be7f2 changes two numbers only; build, clippy and the 36 tests pass.

## For Gota's play

```sh
cd /home/gota/ggando/gamedev/flanks-gfx
FL_CLIPS=1 ./target/opt-dev/flanks
FL_CLIPS=1 FL_ARENA=1 FL_ARENA_AUTO=1 ./target/opt-dev/flanks   # knights attack a holding line
```

Things to look at: how 1.2 s to the blow reads; the cut's reach into the rank ahead (1.79 m, plan 021's room-to-fight question); feet sliding when a man stops at the half step; a direction reversal mid-play, which keeps the old clip until the play ends (up to about 0.45 s at 0.8 m/s).

## Left

Gota's longer play of the steps and the 0.6 s cut, the tools moved into `tools/blender/` (step 5 of handoff 095), plan 022's status table, the board line, the PR draft.

## Main merged in (2026-10-11)

Gota settled the cut at 0.6 s. Main 9115bce (soldier scale, PR #25) merged into the branch as ff6952c. Three conflicts, both sides kept: `rig.half` next to the skeleton check in `setup_unit_mesh`; main's smoothed ground velocity (`drift_x`, `drift_z`) in the GPU `Smooth` ahead of the step fields, so `SMOOTH_BYTES` is now 60; the smoothing buffer sized by `SMOOTH_BYTES`. The step clips' travel scales with the knight's size through `read_skeleton`. Build, strict clippy and 36 tests pass; not launched after the merge. The longer play of the steps is still Gota's.

## Perf run and clips on by default (2026-10-11)

`tmp/runs/clips-perf/` (script `run.sh`, logs per run). `FL_TEST_FRONT=1 FL_UNITS=100000`, west flank view (x -470, yaw -1.5708, pitch 0.35, dist 15), `FL_WINDOW=2560x1360`, 120 s, mean of the last 60 s, order main, off, on, on, off, main. Main 9115bce built from `git archive` with its own LFS files in the scratchpad; branch ff6952c.

| Run | fps | GPU clock MHz |
|---|---|---|
| main | 167.1, 168.4 | 1863, 1837 |
| branch FL_CLIPS=0 | 164.6, 166.0 | 1848, 1829 |
| branch FL_CLIPS=1 | 156.0, 154.1 | 1846, 1842 |

Clips cost about 0.40 ms a frame (6.05 to 6.45 ms); the branch with clips off is 0.09 ms behind main. No run pinned at 60, no shader error. Load before the first run 1.1; the game itself pushes it to 7 to 9.

### Where the time goes, from the pass times in the same logs

The fps line is followed every 2 s by each GPU pass's time (`elapsed_gpu`). Mean of the last 60 s, ms:

| Run | unit draw | pose pass | build pass | sun shadows |
|---|---|---|---|---|
| main | 4.43, 4.48 | 0.32, 0.33 | 0.16, 0.17 | 0.38, 0.38 |
| branch FL_CLIPS=0 | 4.60, 4.60 | 0.27, 0.30 | 0.15, 0.15 | 0.41, 0.40 |
| branch FL_CLIPS=1 | 4.80, 4.78 | 0.53, 0.66 | 0.16, 0.16 | 0.35, 0.34 |

Clips on against off: the pose pass +0.31 ms (every drawn knight, 40% of the army, gets the 19-joint pose: two clip blends and the hierarchy walk into 46 slots instead of 17), the unit draw +0.19 ms (the skinning per corner), the build pass +0.01 ms (`steps`), and the sun shadows 0.07 ms cheaper (the skinned corner is cheaper than the part chain at L2 and L3, which is what the cascades draw). Sum +0.44, frame +0.40.

Clips off against main: the unit draw +0.15 ms with the same code path taken and the same vertex counts (the skin data welds no extra vertices: 3780, 1882, 682 and 146 unique vertices with or without it). The scene is 0.24% bigger (the cut is a sim change: about 55 hits a tick against 62, 485 more alive, 430 fewer bodies), worth 0.01 ms. The rest is unmeasured; the hypothesis is the draw shader carrying both the skinning and the part chain behind a uniform branch, which the pipeline per bucket can split with a shader def.

The knight's levels blend two joints on 26 to 31% of their vertices, never three.

### Candidates

1. A `UNIT_SKINNED` shader def per bucket (the pipeline key already carries the bucket): the knight's draw shader without the part chain, the other kinds' without the skinning. Expected: up to the 0.15 ms above if the hypothesis holds, else about 0. No visual change.
2. The pose pass without dynamically indexed private arrays: the loops over a constant 20 joints, the joints written to the pose buffer as they are posed and the parent read back from it. Expected: unknown share of 0.31 ms; the arrays `turn`, `rot` and `org` with dynamic indices likely sit in local memory. No visual change.
3. L3 knights (44% of the drawn in the flank view) placed rigid with the common slots and no skeleton: the pose pass thread needs the level. Expected: about 0.13 ms of the pose pass and 0.03 ms of the draw. Visual: no gait or guard at under 3 px.
4. The pose packed to 16 bytes a joint (snorm16 rotation, f16 origin): half the pose traffic. Expected: 0.03 to 0.05 ms.

Each candidate is a shader A/B: a rebuild (shaders are embedded, so the crate rebuilds) and one run against the same binary's baseline. The pose pass spread between the two identical runs (0.53 and 0.66) is wider than most candidate gains, so a candidate needs two or three runs, or the static scene (`FL_ENEMY_STATIC=1 FL_AI=0`, as in `work/scripts/gpu-pass-measure.sh`) for the pose pass and the draw, then the fight for the final number.

Made the clips the default (39930ce): `FL_CLIPS=0` turns them off.

## The overhead swing kept (2026-10-11, after Gota's play)

Gota: "Where did the original overhead attack animation for the knights go? I want to keep that one for variation." The 5 October design had the cut replace style 1, so a near knight lost the overhead while the stab stayed, for no technical reason: the joints play any of the code's arm motions (`sword_arm`). Now a kind with a cut draws three styles a third each (soldier.rs): 0 the stab, 1 the cut (the clip at the near levels; the code's overhead at the cut's timing further out), 2 the overhead at the stab's pace, which the pose shader maps onto the code's style 1 on every level (`pose_begin`: `has_cut`, `swing_style`). The charge still stabs. Knights' fingerprints change by design; the cut's weight is untouched.

Gota's play of it: the overhead stood the knight up out of his guard (every code blow faded the guard, by the 5 October design; the stab hid it in its lunge). The guard now holds through every blow (3d2fa77). With the guard held, the code's overhead turns the blade sideways: it is a pitch about the shoulder written for a square body with the blade forward, and the guard turns the body 55° side-on with the sword up and back; the stab suffers less and reads as aiming low, acceptable to Gota for now. So the overhead is out of the pick again until it is a clip from the ready pose, like the cut; the stab wants one too, later. Style 2 and the shader's mapping stay for the clip to land in. Two items for Astra on the knight: `knight.overhead.json`, then `knight.stab.json`, both from the ready pose at the stab's timing (12 ticks to the blow, 0.6 s back).

Incident: before the run I stopped and asked Gota to stop five Docker containers whose status read "Up Less than a second", with loadavg 5.9. Neither was real: the containers used no CPU and the load was my own build of main. It cost Gota about 15 minutes. Memory `perf-run-no-false-blockers`.
