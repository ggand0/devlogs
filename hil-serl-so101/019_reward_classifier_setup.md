# Reward Classifier Setup for HIL-SERL

## Overview

Binary image classifier to detect task success/failure from gripper camera. Used as learned reward function for HIL-SERL fine-tuning.

## Architecture

- **Encoder**: ResNet10 (frozen, from `helper2424/resnet10`)
- **Spatial Embedding**: Learned 4x4 spatial features → 8 dims per channel
- **Classifier Head**: Linear → Dropout → LayerNorm → ReLU → Linear(1)
- **Output**: Binary cross-entropy loss, sigmoid probability

## Key Configuration

```json
{
  "policy": {
    "type": "reward_classifier",
    "model_name": "helper2424/resnet10",
    "image_size": 128,
    "num_cameras": 1,
    "num_classes": 2,
    "hidden_dim": 256,
    "latent_dim": 256
  },
  "dataset": {
    "repo_id": "gtgando/so101_pick_lift_cube_rl_labeled",
    "root": "/home/gota/.cache/huggingface/lerobot/gtgando/so101_pick_lift_cube_rl_labeled",
    "video_backend": "pyav"
  }
}
```

## Image Size Constraint

The `SpatialLearnedEmbeddings` layer expects 4x4 spatial output from ResNet10:

| Input Size | ResNet10 Output | Compatible |
|------------|-----------------|------------|
| 84x84      | 3x3             | No         |
| 128x128    | 4x4             | Yes        |

Images are resized in the model via `_resize_images()` using bilinear interpolation.

## Dataset Labeling

Per-episode success frame cutoffs (frames >= cutoff are labeled as success):

```python
cutoffs = {0: 53, 1: 36, 2: 43, 3: 40, 4: 40, 5: 44, 6: 44, 7: 39, 8: 43, 9: 34}
```

Result: 584 success frames, 416 failure frames (1000 total).

Labels stored in `next.reward` column: 0.0 = failure, 1.0 = success.

## HuggingFace Dataset Sync - Data Loss Risk

**Critical**: `LeRobotDataset()` with a `repo_id` will sync with HuggingFace Hub and can **overwrite local data**.

### What happened

1. Recorded 14 episodes locally
2. Called `LeRobotDataset(repo_id="gtgando/so101_pick_lift_cube")`
3. HF Hub had an older 26-episode version
4. Local 14-episode recording was overwritten with 26-episode version

### Prevention

1. **Use explicit `root` path** pointing to local directory:
   ```python
   dataset = LeRobotDataset(
       repo_id="gtgando/my_dataset",
       root="/home/user/.cache/huggingface/lerobot/gtgando/my_dataset"
   )
   ```

2. **Use unique repo names** for new recordings to avoid conflicts with existing HF repos

3. **Copy dataset before modifications**:
   ```bash
   cp -r ~/.cache/huggingface/lerobot/gtgando/original \
         ~/.cache/huggingface/lerobot/gtgando/original_labeled
   ```

4. **Don't call `push_to_hub()`** until you're certain about the data

### Safe pattern for labeled datasets

```bash
# 1. Copy original dataset
cp -r ~/.cache/huggingface/lerobot/gtgando/so101_pick_lift_cube_rl \
      ~/.cache/huggingface/lerobot/gtgando/so101_pick_lift_cube_rl_labeled

# 2. Modify the copy (add labels to parquet files)

# 3. Use the labeled copy with explicit root
dataset = LeRobotDataset(
    repo_id="gtgando/so101_pick_lift_cube_rl_labeled",
    root="/home/gota/.cache/huggingface/lerobot/gtgando/so101_pick_lift_cube_rl_labeled"
)
```

## Training Command

```bash
cd /home/gota/ggando/ml/lerobot
uv run python -m lerobot.scripts.train \
    --config_path /home/gota/ggando/ml/so101-playground/train_classifier_config.json
```

## Output

- Checkpoints: `outputs/reward_classifier_so101/checkpoints/`
- Logs: `outputs/reward_classifier_so101/`

## Common Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| `Transform 'Resize' is not valid` | LeRobot only supports ColorJitter, SharpnessJitter | Use `image_size` in policy config instead |
| `tensor size mismatch (3 vs 4)` | Wrong spatial dimensions | Use 128x128 input (not 84x84) |
| `torchcodec` error on ROCm | ROCm incompatibility | Set `video_backend: "pyav"` |
| Dataset has wrong episode count | HF sync overwrote local | Use explicit `root` path |
