# Reward Classifier v4 Training

**Date:** 2026-01-27

## Dataset

Combined all four data sources into `reward_v4`:

| Source | Episodes | Frames | Success |
|--------|----------|--------|---------|
| grasp_only_positive | 15 | 991 | 404 |
| grasp_only_negative | 10 | 936 | 100 |
| nighttime | 10 | 679 | 168 |
| daytime_negative | 5 | 999 | 3 |
| **Total** | **40** | **3605** | **675 (18.7%)** |

Train/Val split: 34 train episodes (3022 frames), 6 val episodes (583 frames)

## Training Results

- **Best model:** step 500
- **Best val_loss:** 0.2645
- **Best val_acc:** 92.3%
- **Early stopping:** step 3000 (patience 10)

Final training metrics at step 3000:
- Train loss: 0.0170
- Train acc: 100.0%
- Val loss: 0.3400
- Val acc: 93.0%

## Model Location

```
outputs/reward_classifier_grasp_only_v4/best_model
```

## Dataset Creation

```bash
python scripts/create_grasponly_reward_dataset.py --config reward_v4
```

## Training Command

```bash
uv run --project /home/gota/ggando/ml/lerobot python scripts/train_reward_classifier.py \
    --config configs/reward_classifier_grasponly_v4_train_config.json
```

## Refactoring

Refactored dataset creation to use JSON configs in `data/labels/`:

- Source labels: `grasp_only_positive.json`, `grasp_only_negative.json`, `nighttime.json`, `daytime_negative.json`
- Merge configs: `reward_v1.json`, `reward_v3.json`, `reward_v4.json`
- Documentation: `DATASETS.md`

## Notes

- Model selected based on lowest validation loss
- Overfitting observed (train acc 100% vs val acc 93%)
- Early stopping prevented further overfitting
- Best model was at step 500, much earlier than stopping point
