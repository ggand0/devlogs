# Devlog 038: Image RL with Camera Fix (v11 Reward)

## Experiment Summary

First image-based RL training run after fixing the wrist camera position.
Camera was moved from back side (facing arm base) to front side (facing cube).

## Training Configuration

| Parameter | Value |
|-----------|-------|
| Run dir | `runs/image_rl/20260101_125922/` |
| Reward version | v11 |
| Curriculum stage | 3 (near cube) |
| Steps trained | ~1.92M |
| Camera | Fixed (front-facing) |

## Results

| Steps | Eval Reward | Success Rate |
|-------|-------------|--------------|
| 0k | 21.36 | 0% |
| 160k | 209.13 | 0% |
| 480k | 313.85 | 0% |
| 960k | 315.55 | 0% |
| 1440k | 335.19 | 0% |
| 1920k | 321.58 | 0% |

**Best snapshot:** ~1.3M steps (eval reward ~335)

## Observed Behavior

Agent learned a new exploit at ~1.76M steps:
- Presses cube edge with static finger
- Rotates/tilts cube at an angle
- Supports cube against static finger
- Does NOT attempt proper two-finger grasp

This is similar to the V5 "nudge exploit" documented in devlog 009.

## Root Cause Analysis

The v11 reward has an ungated binary lift bonus:
```python
# Binary lift bonus - NOT gated on is_grasping
if cube_z > 0.02:
    reward += 1.0
```

Cube resting height is ~0.015m. Tilting cube 45 degrees raises center to ~0.021m, triggering the +1.0 bonus without grasping.

**Exploit reward:** ~1.9/step (reach + binary lift)
**Proper grasp reward:** ~2.33/step

The 0.43/step difference isn't enough incentive to learn grasping.

## Fix: v13 Reward

Created v13 reward that gates binary lift bonus on `is_grasping`:
```python
if is_grasping:
    reward += 0.25
    # Continuous lift reward when grasping
    lift_progress = max(0, cube_z - 0.015) / (self.lift_height - 0.015)
    reward += lift_progress * 2.0
    # Binary lift bonus (NOW GATED on is_grasping)
    if cube_z > 0.02:
        reward += 1.0
```

**v13 exploit reward:** ~0.9/step (only reach)
**v13 proper grasp reward:** ~2.33/step

Gap increased from 0.43 to 1.43/step.

## Files

- Training run: `runs/image_rl/20260101_125922/`
- Eval videos: `runs/image_rl/20260101_125922/eval_videos/`
- Best snapshot: `runs/image_rl/20260101_125922/snapshots/best_snapshot.pt`
- TB logs: `runs/image_rl/tb_logs/drqv2_lift_s3/`

## Next Steps

1. Train with v13 reward (binary lift gated on grasp)
2. Monitor for new exploits
3. If v13 fails, consider:
   - Increasing grasp bonus from 0.25 to 0.5
   - Adding cube orientation penalty
   - Curriculum from stage 1 or 2
