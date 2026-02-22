# Reward Classifier Dataset Setup for Reach-and-Grasp Task

## Task Overview

Setting up a simpler PoC task for HIL-SERL: **reach and grasp a red cube** (without lifting). This is easier than pick-and-lift because:
- Shorter horizon (fewer steps to success)
- Clearer intermediate progress
- Less precision needed (no lift stability required)

## Why Reward Classifier

Training DrQ-v2 from scratch with terminal-only rewards failed (~50 intervention episodes, robot exhibits shaking, doesn't learn). Root cause: sparse terminal reward (reward=1 only at episode success) is insufficient for Q-learning from scratch.

Solution: Train offline binary reward classifier to provide dense per-step rewards during RL training.

## Dataset Details

**Repository**: `gtgando/so101_reach_grasp_cube_reward`

| Metric | Value |
|--------|-------|
| Episodes | 15 |
| Total frames | 1296 |
| Success frames | 537 |
| Failure frames | 759 |
| FPS | 10 |
| Image size | 84x84 |

### Success Frame Labeling

Success frames were manually labeled per episode based on when gripper actually grasps the cube (not just when success key was pressed):

| Episode | Success Start | Env Reset Start | Notes |
|---------|---------------|-----------------|-------|
| 0 | 63 | 119 | |
| 1 | 63 | 116 | |
| 2 | 61 | 109 | |
| 3 | 75 | 115 | |
| 4 | 67 | 129 | Gap: 78-88 are failure |
| 5 | 61 | 90 | |
| 6 | 57 | 85 | |
| 7 | 45 | 78 | |
| 8 | 55 | nan | No reset frames |
| 9 | 43 | 71 | |
| 10 | 43 | 67 | |
| 11 | 27 | 48 | |
| 12 | 32 | 68 | |
| 13 | 41 | 63 | |
| 14 | 30 | 53 | |

Labels saved to: `data/so101_reach_grasp_cube_reward_success_frames.csv`

## Bug Fixes During Recording

### 1. Episode Success Spam
**Problem**: After pressing success key, "Episode success" log_say was triggered every step.
**Fix**: Clear keyboard events after reading in step wrapper.
```python
if success:
    self.keyboard_events["episode_success"] = False
```

### 2. Image Data Type Error
**Problem**: Dataset writer expected uint8 [0,255] or float [0,1], but got float [0,255].
**Fix**: Convert images to uint8 before adding to dataset.
```python
if "image" in k and v.dtype == torch.float32:
    v = v.clamp(0, 255).to(torch.uint8)
```

### 3. Leader Arm Locked After Reset
**Problem**: Leader arm was reset to fixed position with torque enabled, user couldn't move it.
**Fix**: Disable torque on leader after follower reset.
```python
if hasattr(self.env, "robot_leader"):
    self.env.robot_leader.bus.sync_write("Torque_Enable", 0, num_retry=3)
```

### 4. IK Reset Auto-Enabled
**Problem**: IK reset was auto-enabled when mujoco_model_path was present.
**Fix**: Made `use_ik_reset` configurable (default false).

## Data Cleaning

### 1. Excluded Environment Reset Frames
Reset motion frames at end of episodes were excluded from dataset:
- Original: 1498 frames
- After exclusion: 1296 frames

### 2. Fixed Video Rotation
**Problem**: Camera config had `rotation: 180` which was incorrect. Static finger appeared on wrong side.
**Fix**:
- Re-encoded all videos with `hflip,vflip` to correct orientation
- Changed config to `rotation: 0` for future recordings

### 3. Updated Reward Labels
Rewards updated based on manual success frame labeling:
- Frames before `success_start`: reward=0
- Frames from `success_start` to `env_reset_start-1`: reward=1
- Episode 4 special case: 67-77 and 89-128 are success, 78-88 are failure

### 4. Updated Meta Files
- `meta/info.json`: Updated `total_frames` from 1498 to 1296
- `meta/episodes_stats.jsonl`: Updated frame counts per episode

## Config Files

### Recording Config
`configs/reward_classifier_record_config.json`:
- `number_of_steps_after_success`: 30
- `normalize_images`: false
- `rotation`: 0 (fixed)
- `fixed_reset_joint_positions`: [-0.4, -104.84, 97.05, 44.09, 89.45, 4.35]

### Training Config
`configs/reward_classifier_train_config.json`:
- `model_name`: helper2424/resnet10
- `num_classes`: 2 (binary)
- `hidden_dim`: 256
- `learning_rate`: 1e-4
- `batch_size`: 32
- `steps`: 2000

## Training Results

### Training Command
```bash
uv run python -m lerobot.scripts.train --config_path configs/reward_classifier_train_config.json
```

### Training Progress
| Step | Loss | Grad Norm |
|------|------|-----------|
| 50 | 0.523 | 1.540 |
| 500 | 0.051 | 0.432 |
| 1000 | 0.027 | 0.335 |
| 1500 | 0.020 | 0.276 |
| 2000 | 0.018 | 0.271 |

### Evaluation Results
**Accuracy: 99.92%** (1295/1296 samples correct)

Target was >95%, achieved 99.92%.

### Trained Model Path
```
outputs/reward_classifier_reach_grasp/checkpoints/002000/pretrained_model
```

### Dataset Fixes During Training
1. Fixed `meta/episodes.jsonl` with correct episode lengths from parquet files
2. Fixed global indices in parquet files to be contiguous (0 to 1295)
3. Regenerated `meta/episodes_stats.jsonl` with proper stats format
4. Changed `image_size` from 84 to 128 (ResNet spatial output compatibility)

## RL Training Config

Created `configs/reach_grasp_rl_env_config.json` with:
- `mode: null` (training mode, not recording)
- `reward_classifier_pretrained_path` pointing to trained classifier
- `resize_size: [128, 128]` (matches classifier training)

## Next Steps

1. Start learner:
```bash
uv run python -m lerobot.scripts.rl.learner --config_path configs/reach_grasp_rl_train_config.json
```

2. Start actor:
```bash
uv run python -m lerobot.scripts.rl.actor --config_path configs/reach_grasp_rl_env_config.json
```

3. Monitor training with wandb
