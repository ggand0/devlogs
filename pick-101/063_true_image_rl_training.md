# 063: True Image-Based RL Training (No Privileged State)

## Summary

Retraining DrQ-v2 after fixing the privileged state bug (devlog 062). Model now has no access to cube_pos - must infer from pixels.

## Observation Space (Fixed)

| Component | Dims | Source |
|-----------|------|--------|
| rgb | (3, 3, 84, 84) | Wrist camera (3 frames) |
| low_dim_state | (3, 18) | Proprioception only |

Proprioception breakdown:
- joint_pos (6) - encoder readings
- joint_vel (6) - encoder derivatives
- gripper_xyz (3) - forward kinematics
- gripper_euler (3) - forward kinematics

**No cube_pos** - must be inferred from vision.

## Mid-Training Results (700k steps)

| Metric | Value |
|--------|-------|
| Steps | 700k |
| Time | 2h 46m |
| Success Rate | 10% |
| Mean Reward | 884 |
| Episode Length | 366 |

## Learning Curve Comparison

| Milestone | Buggy (with cube_pos) | Fixed (no cube_pos) |
|-----------|----------------------|---------------------|
| Grasping attempts | ~100k | ~100k |
| Consistent grasping | ~200k | ~200k |
| First lifts | ~600k | ~600k |
| 10% success | ~700k | 700k |

**Key finding**: Learning curve is nearly identical despite removing cube_pos.

## Hypothesis

The model wasn't relying heavily on cube_pos because:
1. Visual encoder already learned cube localization from pixels
2. cube_pos was redundant information
3. Proprioception (gripper pose) + images sufficient for the task

This is positive for sim-to-real - visual features are meaningful.

## Training Command

```bash
MUJOCO_GL=egl uv run python src/training/train_image_rl.py \
    --config configs/image_based/drqv2_lift_s3_v19.yaml
```

## Checkpoint

```
runs/image_rl/20260106_221053/
├── snapshots/
│   ├── 700000_snapshot.pt  # 10% success
│   └── latest_snapshot.pt
└── eval_videos/
```

Verified no cube_pos in checkpoint:
```
low_dim_obs.0.weight: torch.Size([50, 54])  # 54 = 18 × 3 frames
```

## 800k Results - SUCCESS

| Metric | Value |
|--------|-------|
| Steps | 800k |
| Time | 3h 10m |
| **Success Rate** | **100%** |
| Mean Reward | 239.77 |
| Episode Length | 50 steps |

**cube_pos was unnecessary.** The model achieves 100% success without it, at the same training step (800k) as the buggy version.

## Conclusion

- cube_pos leak didn't help the model - visual features were sufficient
- True image-based policy achieves 100% success
- Ready for sim-to-real transfer experiments
