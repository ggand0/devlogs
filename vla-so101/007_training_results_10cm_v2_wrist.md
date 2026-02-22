# Training Results — 10cm v2 Dataset (Wrist Cam Only)

Dataset: `gtgando/so101_pick_place_10cm_v2` (81 episodes, 24,252 frames, wrist cam only @ 640×480)

Overhead cam excluded via `--exclude-cameras overhead` (dead camera produced black frames from ep 71+).

## SmolVLA Fine-tuning

- **Steps**: 20,000 (batch_size=64) → 1.28M samples, ~53 epochs
- **Duration**: 9h 58m
- **Step time**: ~1.79s/step (updt_s: ~1.19, data_s: ~0.55)
- **Final loss**: 0.006 (L1)
- **Final lr**: 2.7e-05 (cosine decay from 1e-4)
- **Gradient norm**: ~0.12
- **Checkpoints**: 5k, 10k, 15k, 20k (865MB each)
- **Output**: `outputs/train/smolvla_so101_10cm_v2_wrist/`
- **Image preprocessing**: `resize_with_pad` to 512×512 (aspect ratio preserved, padded)
- **Single camera**: wrist only (RealSense D405)

## Comparison with v1 (two cameras)

| | v1 (2 cams, 75 eps) | v2 wrist (1 cam, 81 eps) |
|---|---|---|
| Final loss | 0.006 | 0.006 |
| Epochs | ~57 | ~53 |
| Step time | 1.73s | 1.79s |
| Update time | 1.19s | 1.19s |
| Data loading | 0.53s | 0.55s |
| Gradient norm | ~0.12 | ~0.12 |

Loss converged to the same value. Slightly more data loading time likely due to larger dataset (81 vs 75 episodes). Update time identical — single camera doesn't significantly change forward/backward cost since SmolVLA processes images through a shared vision encoder.

## Dataset Notes

- New gripper + recalibrated arm (old calibration invalidated)
- Mixed lighting: ~47 episodes at night, ~34 episodes during daytime
- 6 failed episodes removed (jerky movements)
- Overhead cam footage exists in dataset but excluded from training

## Eval Results

### SmolVLA 20k — 2026-02-19

- **Checkpoint**: `smolvla_so101_10cm_v2_wrist/checkpoints/020000/pretrained_model`
- **Task**: Pick up the cube and place it in the bowl (10cm range)
- **Episode duration**: 10s, 30fps

| Run | Flags | Success Rate |
|---|---|---|
| 1 | `--no-overhead` | 20% (1/5) |
| 2 | `--no-overhead --direct-reset` | 80% (4/5) |
| 3 | `--no-overhead --direct-reset` | 20% (1/5) |

High variance across runs (20-80%). Failure mode: sometimes grasps air, missing the cube entirely.

### Possible cause: inconsistent demonstrations

During recording, used a tilt trick (nudge cube with static gripper finger to get a good rotation angle before grasping). This was applied even to cube poses that didn't need it, which may confuse the policy — it learns a tilt motion that's unnecessary or counterproductive for some poses, leading to misaligned grasps.

### Lesson: grasp strategy for future recordings

Always use **wrist roll alignment**, never nudge. The parallel gripper can grasp the cube at any rotation as long as the fingers align with opposite faces. Consistent strategy: lower arm above cube → rotate wrist roll to match cube angle → descend → grasp. This gives the policy one clean strategy to learn for every cube pose, instead of a conditional nudge-or-not decision that varies per demonstration.
