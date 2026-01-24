# Reward Classifier v3 - Merged Dataset

## Summary

Extended reward classifier training dataset by merging v1 (20 episodes) with v2 (5 new episodes). Retrained classifier on 25 total episodes with 2586 frames.

## Dataset Details

### v1 Dataset
- 20 episodes, 1932 frames
- 869 success, 1063 failure

### v2 Dataset (New)
- 5 episodes, 654 frames
- Labels from `data/reward_classifier_labels_v2.txt`:
  - Episode 0: frames 49+ success (53 success, 48 fail)
  - Episode 1: frames 107+ success (44 success, 106 fail)
  - Episode 2: frames 67+ success (46 success, 66 fail)
  - Episode 3: all fail (150 fail)
  - Episode 4: frames 98+ success (44 success, 97 fail)
- Total: 187 success, 467 fail

### Merged Dataset
- Path: `/home/gota/.cache/huggingface/lerobot/gtgando/so101_reach_grasp_cube_reward_merged`
- 25 episodes, 2586 frames
- ~1056 success, ~1530 failure (40.8% success rate)

## Training

```bash
uv run python -m lerobot.scripts.train --config_path configs/reward_classifier_train_config.json
```

### Config Changes
- Dataset: `so101_reach_grasp_cube_reward_merged`
- Output: `outputs/reward_classifier_reach_grasp_v3`
- Steps: 3000
- Image size: 128x128 (center cropped from 640x480)

### Results
- Final loss: 0.048
- Model saved to: `outputs/reward_classifier_reach_grasp_v3/checkpoints/003000/pretrained_model`

## Config Updates

Updated `configs/reach_grasp_hilserl_train_config.json`:
- `dataset.repo_id`: `so101_reach_grasp_cube_reward_merged`
- `reward_classifier_pretrained_path`: `outputs/reward_classifier_reach_grasp_v3/checkpoints/003000/pretrained_model`

## Files Changed

- `configs/reward_classifier_train_config.json` - merged dataset
- `configs/reach_grasp_hilserl_train_config.json` - v3 classifier + merged dataset
- `configs/reward_classifier_record_5ep_config.json` - new 5-episode recording config
