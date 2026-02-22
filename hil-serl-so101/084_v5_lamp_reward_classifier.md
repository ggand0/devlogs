# Devlog 084: V5 Lamp Reward Classifier

**Date:** 2025-02-05

## Summary

Recorded new datasets with improved lighting (Yamada Z-LIGHT Z-10R desk lamp) and trained a reward classifier achieving 97.5% validation accuracy. Fixed crop preprocessing mismatch between training and inference.

## Datasets Recorded

### Positive Samples: `gtgando/so101_grasp_only_v3_lamp`
- **Episodes:** 25 recorded, 23 used (skipped 19, 23 as failures)
- **Duration:** 8 seconds per episode
- **Lighting:** Desk lamp clamped above work area for consistent illumination
- **Randomization:** ±1cm in XY and Z (matching HIL-SERL paper)
- **Location:** `/home/gota/.cache/huggingface/lerobot/gtgando/so101_grasp_only_v3_lamp`

### Additional Positive Samples: `gtgando/so101_grasp_only_v3_lamp_positive_5ep`
- **Episodes:** 5 recorded, 4 used (skipped ep4)
- **Duration:** 20 seconds per episode
- **Labels:** Multiple success ranges per episode (not just single success_start)
- **Location:** `/home/gota/.cache/huggingface/lerobot/gtgando/so101_grasp_only_v3_lamp_positive_5ep`

### Negative Samples: `gtgando/so101_grasp_only_v3_lamp_negative`
- **Episodes:** 10
- **Duration:** 20 seconds per episode (longer to capture varied failure modes)
- **Content:** Intentional failed grasps - gripper closing on air, near misses, etc.
- **Location:** `/home/gota/.cache/huggingface/lerobot/gtgando/so101_grasp_only_v3_lamp_negative`

### Additional Mixed Samples: `gtgando/so101_grasp_only_v3_lamp_negative_5ep`
- **Episodes:** 5
- **Duration:** 10 seconds per episode
- **Labels:** Mixed success/fail to prevent reward hacking (ep0: 14-28, ep2: 12-end success; others all fail)
- **Location:** `/home/gota/.cache/huggingface/lerobot/gtgando/so101_grasp_only_v3_lamp_negative_5ep`

## Reward Classifier Dataset

### `gtgando/so101_grasp_only_reward_v5_lamp`
- **Total Episodes:** 42 (23 + 4 positive + 10 + 5 negative/mixed)
- **Total Frames:** 4731
- **Success Frames:** 1034 (21.9%)
- **Failure Frames:** 3697 (78.1%)
- **Location:** `/home/gota/.cache/huggingface/lerobot/gtgando/so101_grasp_only_reward_v5_lamp`

Label configs:
- `data/labels/v3_lamp_positive.json` - success_start frame for each positive episode
- `data/labels/v3_lamp_positive_5ep.json` - success ranges for additional positive episodes
- `data/labels/v3_lamp_negative.json` - all episodes labeled as failure
- `data/labels/v3_lamp_negative_5ep.json` - mixed success/fail labels
- `data/labels/reward_v5_lamp.json` - combines all sources

## Training Results

**Best Model:** Epoch 43, Step 2666
- **Validation Loss:** 0.0674
- **Validation Accuracy:** 97.3%
- **Early Stopping:** Triggered at epoch 53 (patience=10)

Training config: `configs/reward_classifier_grasponly_v5_lamp_train_config.json`

Model saved to:
- `outputs/reward_classifier_grasp_only_v5_lamp/best_model`
- `outputs/reward_classifier_grasp_only_v5_lamp/final_model`

## Crop Preprocessing Fix

Initial training was missing `crop_params_dict`, causing mismatch between training (no crop) and inference (center crop). Fixed by adding to training config:

```json
"crop_params_dict": {
    "observation.images.gripper_cam": [0, 80, 480, 480]
}
```

This matches the HIL-SERL training config and live preview script: 640x480 → crop to 480x480 → resize to 128x128.

## Configuration Updates

Updated to use new reward classifier:
- `configs/grasp_only_hilserl_train_config.json`
- `configs/grasp_only_hilserl_eval_config.json`
- `scripts/reward_classifier_live_preview.py`

## Frame Labeling Process

1. Extract frames from videos:
   ```bash
   uv run python scripts/extract_frames_for_labeling.py \
     --dataset /path/to/dataset \
     --output /path/to/dataset/frames_for_labeling \
     --sample-rate 3
   ```

2. Review frames and note success start frame for each episode in `data/reward_classifier_grasponly_v3_lamp.txt`

3. Create label JSON configs mapping episode indices to success_start frames (multiplied by sample_rate=3)

4. Generate reward dataset:
   ```bash
   uv run python scripts/create_grasponly_reward_dataset.py --config reward_v5_lamp
   ```

## Notes

- The root path in training config must be the full dataset path, not the parent directory (LeRobotDataset uses `root` directly when provided)
- Reward values stored as scalar float in parquet (not list format)
- episodes_stats.jsonl copied from source datasets for LeRobotDataset compatibility
