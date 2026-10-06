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
