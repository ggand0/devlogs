# Image-Based RL: Evaluation and Training Resume

## Overview

This document covers the evaluation of the 400k step checkpoint and preparation for resuming training with improved features.

## Training Progress

### Initial Run (20251231_145806)
- **Algorithm**: DrQ-v2 (image-based RL with random shift augmentation)
- **Environment**: Stage 3 curriculum (near cube, grasp and lift)
- **Steps completed**: 400k
- **Reward at 400k**: ~320 (mean)

### Evaluation Results (400k checkpoint)
```
Episodes: 5
Mean Reward: 319.81 +/- 20.19
Success Rate: 0.0%
```

The agent is learning to reach and grasp but hasn't achieved consistent lifts yet. This is expected at 400k steps for stage 3.

## Features Added to RoboBase Fork

### 1. SB3-Style Logging (`external/robobase/robobase/logger.py`)
- Table format output with colored categories (rollout/, time/, train/)
- Metric name mapping (episode_reward -> ep_rew_mean, etc.)
- Uses `tqdm.write()` to avoid disrupting progress bar
- Filters verbose env_info metrics

### 2. Success Rate Tracking (`src/training/so101_factory.py`)
- Added `SuccessInfoWrapper` that converts `is_success` to `task_success`
- RoboBase expects `task_success` key for success rate logging

### 3. Best Model Saving (`external/robobase/robobase/workspace.py`)
- Added `_best_eval_reward` tracking (initialized to -inf)
- Added `save_best_snapshot()` method called after each eval
- Saves to `snapshots/best_snapshot.pt` when reward exceeds previous best

### 4. Evaluation Script (`src/training/eval_checkpoint.py`)
Multi-camera video evaluation tool:
- Virtual camera support (topdown, side, front, iso views)
- Named camera support (wrist_cam)
- 2x2 grid layout for 4 cameras
- Uses hydra to instantiate agent (same as workspace)
- Reports success rate and mean reward

Usage:
```bash
MUJOCO_GL=egl uv run python src/training/eval_checkpoint.py \
    runs/image_rl/20251231_145806/snapshots/400000_snapshot.pt \
    --num_episodes 5 \
    --cameras topdown wrist_cam side iso
```

## Configuration Changes

### Snapshot Frequency (`src/training/cfgs/so101_lift.yaml`)
Changed from iterations to approximate steps:
```yaml
# Before: snapshot_every_n: 50000 (= ~400k steps with 8 envs)
# After:
snapshot_every_n: 6250  # Save every ~50k steps (6250 iters * 8 envs)
```

## Resume Training

To resume from 400k checkpoint with new features:
```bash
MUJOCO_GL=egl uv run python src/training/train_image_rl.py \
    resume_from=runs/image_rl/20251231_145806/snapshots/400000_snapshot.pt
```

This will:
- Continue from step 400k
- Create new run directory with current timestamp
- Save snapshots every ~50k steps
- Save best model based on eval reward
- Log success rate during evaluation

## Technical Notes

### RoboBase Snapshot Structure
Snapshots contain:
- `agent`: Model state dict
- `cfg`: Full OmegaConf config
- `_pretrain_step`, `_main_loop_iterations`, `_global_env_episode`: Training state

### Iterations vs Steps
- RoboBase counts iterations, not env steps
- With 8 parallel envs: `steps ≈ iterations * 8`
- `snapshot_every_n: 6250` iterations ≈ 50k steps

### PyTorch 2.6+ Compatibility
- `torch.load()` now defaults to `weights_only=True`
- Must use `weights_only=False` to load OmegaConf configs from snapshots

## Next Steps

1. Resume training from 400k to 1M steps
2. Monitor success rate during evaluation
3. Best model will be saved automatically
4. Evaluate final model with multi-camera videos
