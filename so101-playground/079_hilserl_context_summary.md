# HIL-SERL Context Summary

## Current State

**Checkpoint:** `outputs/hilserl_grasp_only_v2/checkpoints/003000/pretrained_model`
**Training episodes:** 242
**Eval success rate:** 60% (3/5 episodes)

## Key Files

| File | Purpose |
|------|---------|
| `configs/grasp_only_hilserl_train_config.json` | Training config |
| `configs/grasp_only_hilserl_eval_config.json` | Eval config (random_ee_reset: false) |
| `scripts/hilserl_inference.py` | Inference script for running trained policies |
| `devlogs/077_gripper_action_format_mismatch.md` | Gripper fix documentation |
| `devlogs/078_hilserl_grasp_only_v2_training.md` | Training run documentation |

## Gripper Fix (devlog 077)

**Problem:** Gripper worked in episode 1 but failed to reopen in episode 2+.

**Root cause:** Action space defined gripper as [0, 2] but SAC outputs [-1, 1] (tanh).

**Fixes applied:**
1. `gym_manipulator.py:503-506` - Changed action space bounds from [0,2] to [-1,1]
2. `gym_manipulator.py:2226-2233` - Changed teleop format from [0,1,2] to [-1,0,1]
3. `so101_follower_end_effector.py` - Direct scaling for gripper delta

## Training Configuration

```json
{
  "n_obs_steps": 1,  // No frame stacking
  "utd_ratio": 20,
  "discount": 0.97,
  "temperature_init": 0.01,
  "use_augmentation": true,  // DrQ-style random shifts
  "augmentation_pad": 4,
  "random_ee_reset": true,
  "random_ee_range_xy": 0.03  // ±3cm variation
}
```

## Evaluation Results

### With random_ee_reset: true (5 episodes)
| Episode | Steps | Reward | Result |
|---------|-------|--------|--------|
| 1 | 46 | 118.30 | SUCCESS |
| 2 | 30 | 219.60 | SUCCESS |
| 3 | 100 | 45.25 | FAIL |
| 4 | 30 | 119.10 | SUCCESS |
| 5 | 100 | -5.00 | FAIL |

### With random_ee_reset: false (5 episodes)
| Episode | Steps | Reward | Result |
|---------|-------|--------|--------|
| 1 | 30 | 169.35 | SUCCESS |
| 2 | 100 | -5.00 | FAIL |
| 3 | 38 | 179.00 | SUCCESS |
| 4 | 100 | 15.10 | FAIL |
| 5 | 70 | 348.25 | SUCCESS |

**Both got 60% (3/5)** - position randomization is NOT the issue.

## Known Issues

1. **Inconsistent gripper behavior:** In failed episodes, policy outputs positive gripper values (opening) when it should close.

## Sample Efficiency Discussion

HIL-SERL paper reports 20-50 minutes (~100-300 episodes) for tasks like peg insertion.

At 242 episodes with 60% success, possible factors:
- No BC pretraining (learning from scratch) <- NOTE: the paper does NOT use BC
- Manual cube placement variance each episode
- `n_obs_steps: 1` (no frame stacking vs DrQ-v2's typical 3 frames)
- State includes velocities but image doesn't capture motion

## Architecture Notes

**Observation:**
- `observation.images.gripper_cam`: [1, 3, 128, 128] - cropped/resized from 640x480
- `observation.state`: [1, 18] - joint_pos(6) + joint_vel(6) + ee_xyz(3) + ee_euler(3)

**Action:** [4] - delta_x, delta_y, delta_z, gripper (all in [-1, 1])

**Vision encoder:** `helper2424/resnet10` (frozen)

## Inference Command

```bash
uv run python scripts/hilserl_inference.py \
    --config_path configs/grasp_only_hilserl_train_config.json \
    --checkpoint outputs/hilserl_grasp_only_v2/checkpoints/003000/pretrained_model \
    --num_episodes 5
```

## Next Steps to Try

1. **Continue training** to 5000+ steps
2. **Add frame stacking** (`n_obs_steps: 3`) - requires retraining
3. **Lower temperature** during eval for less exploration
4. **Fix camera retry logic** for blank frame recovery
5. **Investigate failed episodes** - why does policy open gripper when cube is in front?

## Reward Classifier

**Model:** `outputs/reward_classifier_grasp_only_v5/best_model`
- 97.3% val accuracy
- Trained on v5 dataset (50 episodes, 5604 frames)
- Uses same crop `[0, 80, 480, 480]` and resize to 128x128
