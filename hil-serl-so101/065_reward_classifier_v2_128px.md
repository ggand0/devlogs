# Reward Classifier v2 - 128x128px Training

## Overview

Re-recorded and retrained the reward classifier for the reach-and-grasp task at 128x128 resolution (up from 84x84).

## Dataset Recording

### Configuration Changes

**Recording config** (`reward_classifier_record_config.json`):
- `num_episodes`: 20 (matches HIL-SERL paper recommendation of 20-30)
- `resize_size`: null (record at native 640x480)
- `control_time_s`: 15.0 (max episode length)
- `reset_time_s`: 5.0 (env reset duration with audio cue)
- `locked_joints`: [] (all joints free for reach-and-grasp)
- `reward_classifier_pretrained_path`: null (no auto-termination during recording)

### Image Processing Pipeline

1. Camera captures at **640x480** (native resolution)
2. Center crop to **480x480** (removes 80px from left/right)
3. Resize to **128x128** (lanczos scaling)

```bash
ffmpeg -i input.mp4 -vf "crop=480:480:80:0,scale=128:128:flags=lanczos" output.mp4
```

### Dataset Statistics

| Metric | Value |
|--------|-------|
| Episodes | 20 |
| Total frames | 1932 |
| Success frames | 869 |
| Failure frames | 1063 |
| FPS | 10 |
| Resolution | 128x128 |

### Success Frame Labels

Manually labeled success start frame per episode:

```
00: 119    05: 39     10: 47     15: 62
01: 129    06: 76     11: 39     16: 33
02: 43     07: 44     12: 46     17: 35
03: 42     08: 36     13: 57     18: 41
04: 41     09: 42     14: 37     19: 55
```

Frames at index >= success_start have reward=1, earlier frames have reward=0.

## Training Configuration

**Hyperparameters** (`reward_classifier_train_config.json`):

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| model_name | helper2424/resnet10 | Lightweight CNN |
| image_size | 128 | Higher resolution |
| batch_size | 32 | Good for ~2k samples |
| steps | 3000 | Sufficient for convergence |
| learning_rate | 1e-4 | Standard |
| dropout_rate | 0.2 | Regularization for small dataset |
| weight_decay | 0.01 | Prevent overfitting |
| image_transforms | enabled | Data augmentation (brightness, contrast, saturation, hue) |

## Training Results

| Metric | Value |
|--------|-------|
| Final loss | 0.037 |
| Accuracy | **98.86%** (1910/1932) |
| Steps | 3000 |
| Epochs | ~50 |

### Model Path

```
outputs/reward_classifier_reach_grasp_v2/checkpoints/003000/pretrained_model
```

## Code Changes

### Recording Script (`gym_manipulator.py`)

Added audio cues for environment reset timing:
- "Resetting. N seconds." when reset starts
- "Reset done." when reset completes
- 5 second wait for manual object repositioning

### Config Updates

All reach-and-grasp configs updated:
- `locked_joints: []` (was `[3, 4]`)
- `locked_joint_positions: {}` (was `{"3": 90.0, "4": 90.0}`)
- Image resolution: 128x128

## Files

- Dataset: `/home/gota/.cache/huggingface/lerobot/gtgando/so101_reach_grasp_cube_reward_v1`
- Model: `outputs/reward_classifier_reach_grasp_v2/checkpoints/003000/pretrained_model`
- Videos backup (640x480): `videos_backup/`
- Processed videos (128x128): `videos/`
- Extracted frames: `frames_128/`
