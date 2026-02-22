# HIL-SERL Grasp Only v2 Training Run

## Overview

First successful HIL-SERL training run with the gripper action format fix (devlog 077). The policy learned to grasp and re-grasp the cube within episodes.

## Checkpoint

**Path:** `outputs/hilserl_grasp_only_v2/checkpoints/003000/pretrained_model`

**Config:** `configs/grasp_only_hilserl_train_config.json`

## Training Stats (at step 3000)

| Metric | Value |
|--------|-------|
| Episodes | 242 |
| Buffer size | 19,422 |
| Critic loss | 33.50 |
| Actor loss | -28.18 |
| Temperature α | 0.0234 |

## Key Settings

```json
{
  "policy": {
    "type": "sac",
    "vision_encoder_name": "helper2424/resnet10",
    "freeze_vision_encoder": true,
    "utd_ratio": 20,
    "discount": 0.97,
    "temperature_init": 0.01,
    "target_entropy": -2.0,
    "online_buffer_capacity": 200000,
    "offline_buffer_capacity": 5000
  },
  "env": {
    "fps": 10,
    "reward_classifier_pretrained_path": "outputs/reward_classifier_grasp_only_v5/best_model",
    "wrapper": {
      "use_ik_reset": true,
      "ik_reset_ee_pos": [0.25, 0.0, 0.03],
      "random_ee_reset": true,
      "random_ee_range_xy": 0.03,
      "control_time_s": 10.0,
      "reset_time_s": 2.0,
      "reset_delay_s": 2.0
    }
  }
}
```

## Episode Rewards (sample from episodes 236-241)

| Episode | Reward | Intervention % |
|---------|--------|----------------|
| 236 | 5.05 | 0% |
| 237 | 129.15 | 0% |
| 238 | 209.55 | 0% |
| 239 | 45.25 | 0% |
| 240 | 229.65 | 0% |
| 241 | 47.05 | 0% |

Notable: Multiple consecutive episodes with 0% intervention and positive rewards, indicating autonomous grasping.

## Gripper Fix Validation

The gripper action format fix from devlog 077 is working:
- Gripper can close AND reopen during episodes
- Re-grasping works within single episodes
- No more stuck-closed gripper issue from episode 2+

## Hardware Setup

- Robot: SO101 follower with end-effector control
- Camera: Innomaker gripper cam (replacement after original failed)
- Teleop: SO101 leader arm
- Crop: `[0, 80, 480, 480]` center crop to 480x480, resize to 128x128

## Inference

```bash
uv run python scripts/hilserl_inference.py \
    --config_path configs/grasp_only_hilserl_train_config.json \
    --checkpoint outputs/hilserl_grasp_only_v2/checkpoints/003000/pretrained_model \
    --num_episodes 5
```

## Evaluation Results (checkpoint 3000)

5-episode eval with random EE reset enabled:

| Episode | Steps | Reward | Result |
|---------|-------|--------|--------|
| 1 | 46 | 118.30 | ✓ SUCCESS |
| 2 | 30 | 219.60 | ✓ SUCCESS |
| 3 | 100 | 45.25 | ✗ FAIL (timeout) |
| 4 | 30 | 119.10 | ✓ SUCCESS |
| 5 | 100 | -5.00 | ✗ FAIL (timeout) |

**Success rate: 60% (3/5)**

Failed episodes hit the 100-step timeout without the reward classifier detecting a grasp.

## Sample Efficiency Discussion

HIL-SERL paper reports ~20-50 minutes of robot time for tasks like peg insertion and cable routing, translating to roughly 100-300 episodes.

At 242 episodes with 60% success rate, possible factors:

1. **Random EE reset** adds difficulty - paper uses fixed start positions per task
2. **Cube position varies** each episode (2s manual placement delay)
3. **No BC pretraining** - learning from scratch with only online data
4. **Generalization requirement** - policy must handle ±3cm XY variation

The training logs show episodes 236-241 had 0% intervention with positive rewards, suggesting the policy can grasp autonomously but may struggle with certain random start positions.

## Next Steps

- Disable `random_ee_reset` during eval to test fixed-position success rate
- Continue training for better position generalization
- Consider reducing `random_ee_range_xy` from 3cm to 2cm
