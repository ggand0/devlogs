# Handoff: HIL-SERL Reward Classifier Setup

## Current Situation

Training DrQ-v2 from scratch with HIL-SERL is NOT working due to sparse terminal-only rewards. After ~50 intervention episodes, robot exhibits shaking behavior and doesn't learn.

## Root Cause

Terminal-only reward (reward=1 only at episode success) is insufficient for Q-learning from scratch. Need **dense per-step rewards** from a reward classifier.

## What User Wants to Do Now

Train an **offline reward classifier** on existing labeled dataset, then use it during RL training to provide dense rewards.

## Key Files

| File | Purpose |
|------|---------|
| `/home/gota/ggando/ml/lerobot/src/lerobot/policies/sac/reward_model/configuration_classifier.py` | Classifier config |
| `/home/gota/ggando/ml/lerobot/src/lerobot/policies/sac/reward_model/modeling_classifier.py` | Classifier implementation |
| `/home/gota/ggando/ml/so101-playground/outputs/hilserl_drqv2_scratch/train_config.json` | RL training config |

## Available Labeled Datasets

| Dataset | Frames | Location |
|---------|--------|----------|
| `so101_pick_lift_cube_locked_wrist_labeled` | 1,574 | `/home/gota/.cache/huggingface/lerobot/gtgando/so101_pick_lift_cube_locked_wrist_labeled` |
| `so101_pick_lift_cube_rl_labeled` | 1,000 | `/home/gota/.cache/huggingface/lerobot/gtgando/so101_pick_lift_cube_rl_labeled` |

## How Reward Classifier Works

1. Binary classifier trained on images: success (label=1) vs failure (label=0)
2. Outputs continuous probability [0, 1] at every timestep
3. Provides dense reward signal instead of terminal-only

## Training Command (from LeRobot docs)

```bash
lerobot-train --config_path path/to/reward_classifier_train_config.json
```

## Config Structure for Classifier Training

```json
{
  "policy": {
    "type": "reward_classifier",
    "model_name": "helper2424/resnet10",
    "model_type": "cnn",
    "num_cameras": 1,
    "num_classes": 2,
    "hidden_dim": 256,
    "dropout_rate": 0.1,
    "learning_rate": 1e-4,
    "device": "cuda",
    "input_features": {
      "observation.images.gripper_cam": {
        "type": "VISUAL",
        "shape": [3, 84, 84]
      }
    }
  },
  "dataset": {
    "repo_id": "gtgando/so101_pick_lift_cube_locked_wrist_labeled"
  }
}
```

## Using Classifier in RL Training

Add to train_config.json:
```json
{
  "env": {
    "reward_classifier_pretrained_path": "path/to/trained/classifier"
  }
}
```

Or in newer config structure:
```json
{
  "env": {
    "processor": {
      "reward_classifier": {
        "pretrained_path": "path/to/trained/classifier",
        "success_threshold": 0.7,
        "success_reward": 1.0
      }
    }
  }
}
```

## Important Notes

1. Classifier is trained OFFLINE, then FIXED during RL (per HIL-SERL paper)
2. Target accuracy: >95% before using in RL
3. User expressed concern about classifier reliability - research shows "ever-correcting" online updates exist but aren't in LeRobot

## Devlogs Written

- `devlogs/058_hilserl_scratch_training_analysis.md` - Why scratch training fails
- `devlogs/059_bc_loss_experiment_reverted.md` - BC loss experiment (reverted)
- `devlogs/060_reward_classifier_research.md` - Full research on reward classifiers

## Next Steps

1. Create reward classifier training config for `so101_pick_lift_cube_locked_wrist_labeled`
2. Train classifier with `lerobot-train`
3. Evaluate accuracy (need >95%)
4. Add trained classifier path to RL config
5. Restart RL training with dense rewards

## References

- HIL-SERL Paper: https://arxiv.org/html/2410.21845v1
- LeRobot HIL-SERL Docs: https://huggingface.co/docs/lerobot/hilserl
- Ever-Correcting Classifier: https://arxiv.org/html/2512.24288
