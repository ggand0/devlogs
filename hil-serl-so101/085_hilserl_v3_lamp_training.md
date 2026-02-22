# Devlog 085: HIL-SERL v3_lamp Training

**Date:** 2026-02-06

## Summary

Trained HIL-SERL grasp-only policy with improved setup: desk lamp lighting (Yamada Z-LIGHT Z-10R), corrected reset height (z=0.07), and v5_lamp reward classifier (97.3% accuracy). Achieved significantly better performance than previous runs with z=0.03 reset height.

## Training Setup

### Hardware
- SO-101 follower arm with gripper camera
- Yamada Z-LIGHT Z-10R desk lamp for consistent illumination
- Red cube target object

### Key Hyperparameters

| Parameter | Value |
|-----------|-------|
| batch_size | 256 |
| utd_ratio | 20 |
| discount | 0.97 |
| actor_lr | 0.0003 |
| critic_lr | 0.0003 |
| temperature_init | 0.01 |
| target_entropy | -2.0 |
| num_critics | 2 |
| critic_target_update_weight | 0.005 |
| latent_dim | 256 |
| hidden_dims | [256, 256] |
| vision_encoder | helper2424/resnet10 (frozen) |
| image_encoder_hidden_dim | 32 |
| action_scale | 0.02 |
| fps | 10 |
| control_time_s | 10.0 |

### Environment Config
- `ik_reset_ee_pos`: [0.25, 0.0, 0.07] (fixed from previous z=0.03)
- `random_ee_range_xy`: 0.01
- `random_ee_range_z`: 0.01
- `reward_classifier`: v5_lamp (97.3% accuracy)

## Training Results

- **Total Episodes:** 757 (148 + 609 across 2 resumed sessions; first 11-ep run lost to HW issue)
- **Total Steps:** ~48,300
- **Optimization Steps:** 9,500

### Final Performance (last 50 episodes)
- **Mean Reward:** 100.58
- **Mean Intervention:** 5.9%
- **Episodes with 0% intervention (last 100):** 61/100

### Best Episode
- **Reward:** 309.00 (episode 587)

## Training Curves

![Training Curves](../outputs/hilserl_grasp_only_v3_lamp/training_curves.png)

## Key Improvements Over Previous Setup

1. **Lighting:** Desk lamp provides consistent illumination, reducing visual noise
2. **Reset Height:** z=0.07 matches recording data (was incorrectly z=0.03)
3. **Reward Classifier:** v5_lamp trained with proper crop preprocessing (97.3% accuracy)
4. **Randomization:** Reduced to ±1cm (matching HIL-SERL paper recommendations)

## Checkpoints

Latest checkpoint: `outputs/hilserl_grasp_only_v3_lamp/checkpoints/009500/pretrained_model`

## Evaluation Results

Evaluated on 20 episodes (2 runs of 10) with varying cube positions:

| Run | Cube Positions | Success Rate | Mean Reward | Mean Steps |
|-----|----------------|--------------|-------------|------------|
| 1 | Left-biased (center) | 8/10 (80%) | 116.04 | 51.1 |
| 2 | Evenly distributed (edges) | 6/10 (60%) | 78.23 | 63.4 |
| **Combined** | **Mixed** | **14/20 (70%)** | **97.14** | **57.3** |

### Position Sensitivity

- **Strong:** Center positions - policy reliably grasps cubes near workspace center
- **Weak:** Edge positions (top-right, bottom-right, far edges) - struggles with cubes at workspace boundaries

The ±1cm reset randomization during training likely concentrated data near the center, limiting generalization to edge positions.

### Camera Visible Area

The gripper camera's visible area on the table forms a cone-like shape (not a square) due to:
- Camera mounted at an angle looking downward
- Different horizontal/vertical FOV
- Perspective projection onto the table surface

At the z=0.07 reset height, the visible workspace is roughly trapezoidal - narrower near the gripper, wider further away. Edge positions with fewer successful training examples remain challenging for the policy.

## Notes

- Training required ~600 episodes to converge (longer than expected)
- Intervention rate dropped significantly after ~400 episodes
- Some episodes still require intervention for edge cases (cube at workspace boundary)
