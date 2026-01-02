# Devlog 036: Image RL Evaluation Results (1M and 2M Steps)

## Training Configuration
- **Algorithm**: DrQ-v2 (image-based RL)
- **Curriculum Stage**: 3 (gripper near cube, open, needs to grasp and lift)
- **Reward Version**: v11
- **Network**: 32 channels encoder, 256x256 MLP for actor/critic
- **Exploration**: Linear stddev decay 1.0 → 0.1 over 500k steps

## Evaluation Results

| Metric | 1M Steps | 2M Steps |
|--------|----------|----------|
| Mean Reward | 301.85 ± 76.03 | 326.50 ± 15.38 |
| Success Rate | 0% | 0% |
| Best Snapshot | ~400k steps | ~1.45M steps |

## Observed Behavior

### Common Pattern (90% of episodes)
1. Agent nudges cube with static finger toward arm base
2. Supports cube against static finger at arm base
3. Does not attempt proper two-finger grasp

### Rare Attempts (10% of episodes)
- Sometimes tries to grab with both fingers at beginning
- Occasionally attempts grasp at arm base after pushing cube there

## Analysis

### Positive Signs
- Reward consistently increasing (301 → 326)
- Much lower variance at 2M (76 → 15) indicates stable policy
- Agent learned reaching/approach behavior
- Smoothed eval rewards still trending upward

### Problems Identified

1. **Local Optimum**: Agent found exploit of using static finger + arm base as "support"
   - Gets reaching rewards without learning proper grasp

2. **Camera Position Issue**: Wrist camera is on back side (toward arm base)
   - Real-world camera mount is on front side (toward cube)
   - Agent may be learning to push cube toward camera view
   - Current: `pos="0.0 0.055 0.02"` (Y+ is toward arm base in gripper frame)
   - Should be: `pos="0.0 -0.055 0.02"` (Y- is toward cube/front)

3. **Cube Reset Position**: Using same IK reset position for stage 3
   - `cube_x = 0.25 + random(-0.02, 0.02)`
   - `cube_y = -0.10 + random(-0.02, 0.02)`
   - Cube spawns in front of gripper, but camera sees backward

## Camera Position Fix (Completed)

The wrist camera was positioned incorrectly on the back side of the gripper (facing toward arm base).

**Before (incorrect)**:
```xml
<camera name="wrist_cam" pos="0.0 0.055 0.02" euler="0 0 0" fovy="75"/>
```

**After (correct)**:
```xml
<camera name="wrist_cam" pos="0.0 -0.055 0.02" euler="0 0 3.14159" fovy="75"/>
```

Changes:
- Y position: +0.055 → -0.055 (moved from back to front side)
- Added 180° rotation (euler Z = π) to face the gripper/cube area
- Now matches real hardware camera mount position

Visualization test: `runs/camera_test_v2/` contains:
- `wrist_cam.png` - New front-side camera view
- `combined_grid.png` - Multi-view comparison
- `camera_demo.mp4` - Video with random motion

## Training Infrastructure Added

### Custom Workspace (`src/training/workspace.py`)
- `SO101Workspace` extends RoboBase's Workspace
- Auto-plots learning curves every 50k steps during training
- Plots final curves at end of training
- Saves to `{run_dir}/learning_curves.png`

### Visualization Scripts
- `scripts/plot_learning_curves.py` - Plot tensorboard logs manually
- `scripts/visualize_camera.py` - Test camera views with multi-angle rendering

## Next Steps

1. **Retrain with Fixed Camera** (recommended)
   - Visual input completely different with corrected camera
   - Cannot resume from old weights (learned wrong view)
   - Start fresh training with same stage 3 curriculum

2. **Monitor for Local Optimum Avoidance**
   - With correct camera view, agent should see cube properly
   - May naturally learn to approach from front instead of pushing backward

3. **Reward Shaping Considerations** (if still stuck)
   - Current v11 reward may not penalize "pushing" behavior enough
   - Consider adding penalty for cube moving toward arm base
   - Or bonus for cube staying in original position during approach

4. **Curriculum Progression**
   - If stage 3 succeeds, progress to stage 4 (reach + grasp + lift + place)
   - Eventually remove curriculum for full task learning

## Files
- 1M eval videos: `runs/image_rl/20251231_193257/eval_videos/`
- 2M eval videos: `runs/image_rl/20260101_030120/eval_videos/`
- Learning curves: `runs/image_rl/20260101_030120/learning_curves*.png`
- TB logs: `runs/image_rl/tb_logs/drqv2_lift_s3/`
- Camera test: `runs/camera_test_v2/`

## Technical Details

### DrQ-v2 Hyperparameters
| Parameter | Value |
|-----------|-------|
| Frame stack | 3 |
| Image size | 84x84 |
| Encoder channels | 32 |
| Actor/Critic MLP | [256, 256] |
| Batch size | 256 |
| Num train envs | 8 |
| Replay buffer | 100k |
| Exploration stddev | linear(1.0, 0.1, 500k) |
| Action repeat | 2 |
| Eval every | 10k steps |
| Snapshot every | ~50k steps |

### Environment Configuration
| Parameter | Value |
|-----------|-------|
| Episode length | 200 steps |
| Curriculum stage | 3 (near cube) |
| Reward version | v11 (shaped) |
| Lock wrist | false |
| Action space | 5D (xyz + wrist_roll + gripper) |
