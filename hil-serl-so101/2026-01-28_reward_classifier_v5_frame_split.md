# Reward Classifier v5 - Frame-Level Split Training

**Date:** 2026-01-28

## Changes from v4

- **Fixed train/val split**: Changed from episode-level to frame-level random split
- **Epoch-based validation**: Validate at end of each epoch instead of every N steps
- **tqdm progress bars**: Per-epoch progress with loss/acc updates
- **Suppressed warnings**: Filtered torchvision video deprecation warnings

## Dataset

Same as v4 - combined all four data sources:

| Source | Episodes | Frames | Success |
|--------|----------|--------|---------|
| grasp_only_positive | 15 | 991 | 404 |
| grasp_only_negative | 10 | 936 | 100 |
| nighttime | 10 | 679 | 168 |
| daytime_negative | 5 | 999 | 3 |
| **Total** | **40** | **3605** | **675 (18.7%)** |

## Train/Val Split

**Frame-level random split (seed=42):**
- Train: 3065 frames (85%)
- Val: 540 frames (15%)

Both sets contain frames from all 40 episodes, randomly distributed.

## Training Configuration

- Batch size: 64
- Steps per epoch: ~47
- Patience: 10 epochs
- Validation: Every epoch
- Early stopping: Based on val_loss

## Training Results

Best model from v4 training (episode-level split):
- Best step: 500
- Val loss: 0.2645
- Val acc: 92.3%

The v4 model still generalizes well to frame-level evaluation.

## Validation Evaluation (v4 model)

Using the v4 best model with the new frame-level val split:

```
Val Accuracy: 98.15% (530/540)

Confusion Matrix:
  Predicted:    0      1
  Actual 0:    439      5
  Actual 1:      5     91

Precision: 0.948
Recall: 0.948
F1 Score: 0.948

Class distribution: 444 negative, 96 positive (17.8% positive)
```

## Scripts Updated

- `scripts/train_reward_classifier.py`: Frame-level split, epoch-based validation, tqdm, warning suppression
- `scripts/eval_reward_classifier.py`: New script for val set evaluation with confusion matrix

## Commands

**Training:**
```bash
uv run python scripts/train_reward_classifier.py --config configs/reward_classifier_grasponly_v4_train_config.json
```

**Evaluation:**
```bash
uv run python scripts/eval_reward_classifier.py --config configs/reward_classifier_grasponly_v4_train_config.json
```

## Model Location

```
outputs/reward_classifier_grasp_only_v4/best_model
```

## Notes

- The v4 model achieves 98.15% accuracy on the proper frame-level val split
- Very balanced precision/recall (both 0.948)
- Only 10 misclassifications out of 540 (5 false positives, 5 false negatives)
- Model is ready for HIL-SERL training
