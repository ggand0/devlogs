# Reward Classifier v5: Angled Camera Data

## Overview

Added 10 episodes of angled camera data to the reward classifier training dataset. This data was recorded with a new camera after the original Innomaker camera failed.

## Dataset

**V5 = V4 + Angled Camera Data**

| Metric | V4 | Angled | V5 |
|--------|-----|--------|-----|
| Episodes | 40 | 10 | 50 |
| Frames | 3605 | 1999 | 5604 |
| Success | 675 | 1343 | 2018 |
| Fail | 2930 | 656 | 3586 |
| Success % | 18.7% | 67.2% | 36.0% |

**Path:** `/home/gota/.cache/huggingface/lerobot/gtgando/so101_grasp_only_reward_v5`

## Annotation Format

Labels file: `data/labels/angled_v1.json`

```json
{
    "0": {"success_ranges": [[27, 38], [49, 61], [77, 95], ...]},
    "1": {"success_ranges": [[23, null]]},  // null = to end
    ...
}
```

Source annotation: `data/reward_classifier_grasponly_v1_angled.txt`

## Files Created

| File | Purpose |
|------|---------|
| `data/labels/angled_v1.json` | Parsed labels from annotation |
| `scripts/merge_angled_reward_dataset.py` | Merge v4 + angled → v5 |
| `configs/reward_classifier_grasponly_v5_train_config.json` | Training config |

## Training

```bash
uv run python scripts/train_reward_classifier.py --config configs/reward_classifier_grasponly_v5_train_config.json
```

**Config changes from v4:**
- Dataset: v4 → v5
- Steps: 5000 → 6000 (more data)
- Output: `outputs/reward_classifier_grasp_only_v5`

**Training parameters:**
- Batch size: 64
- Steps per epoch: ~88 (5604 / 64)
- Max epochs: ~68 (6000 / 88)
- Early stopping patience: 10 epochs
- Val ratio: 15%

## Training Results

**Best model:** Epoch 26 (step 1924)
- Val loss: 0.0789
- Val accuracy: 97.3%

Early stopping triggered at epoch 36 (patience 10).

**Model path:** `outputs/reward_classifier_grasp_only_v5/best_model`

## Camera Notes

The new camera (same Innomaker model) has slightly different framing - gripper appears more toward the right edge of the 640x480 frame. The center crop `[0, 80, 480, 480]` still works but is at the edge tolerance.

If issues arise, consider:
1. Right-aligned crop `[0, 160, 480, 480]`
2. Physical camera adjustment
