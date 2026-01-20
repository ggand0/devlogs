# 074: Sim2Real Inference Handoff

## Overview

This document provides context for implementing real robot inference in `so101-playground/` using the trained seg+depth policy.

## Model Weights

### 1. Segmentation Model (EfficientViT-B0)

- **Checkpoint**: `pick-101/outputs/efficientvit_seg/best.ckpt`
- **Architecture**: EfficientViT-B0 + FPN decoder (~0.85M params)
- **Input**: RGB image (640x480 or any resolution)
- **Output**: 6-class segmentation mask
- **Val IoU**: 0.811 (640x480), 0.817 (84x84)

### 2. Depth Model (Depth Anything V2 Small)

- **Source**: HuggingFace `depth-anything/Depth-Anything-V2-Small`
- **Input**: RGB image
- **Output**: Disparity map (inverse depth, 0-255 after normalization)
- **Format**: Near=255, Far=0 (matches training)

### 3. Policy Model (DrQV2)

- **Checkpoint**: `pick-101/runs/seg_depth_rl/20260119_192807/snapshots/1200000_snapshot.pt`
- **Success Rate**: 70% (10 eval episodes in sim)
- **Mean Reward**: 359.08 ± 154.77

## Observation Format

The policy expects `(6, 84, 84)` input from frame_stack=3 of `(2, 84, 84)` frames.

### Per-Frame Channels (2 channels)

| Channel | Content | Range |
|---------|---------|-------|
| 0 | Segmentation class IDs | 0-5 (integers) |
| 1 | Disparity (inverse depth) | 0-255 |

### Segmentation Classes

| ID | Class | Description |
|----|-------|-------------|
| 0 | unlabeled | Pixels with no annotation |
| 1 | background | Arm, sky, everything else |
| 2 | table | Ground/floor surface |
| 3 | cube | Target object |
| 4 | static_finger | Fixed gripper finger |
| 5 | moving_finger | Actuated gripper finger |

## Inference Pipeline

```python
# 1. Capture RGB frame from wrist camera (640x480)
rgb = camera.capture()  # (480, 640, 3) BGR

# 2. Run segmentation
seg_mask = seg_model.predict(rgb)  # (480, 640) uint8, values 0-5

# 3. Run depth estimation
disparity = depth_model.predict(rgb)  # (480, 640) float32

# 4. Normalize disparity to 0-255
disparity = (disparity - disparity.min()) / (disparity.max() - disparity.min() + 1e-6)
disparity = (disparity * 255).astype(np.uint8)

# 5. Center crop to square (480x480 from 640x480)
h, w = 480, 640
crop_x = (w - h) // 2  # 80
seg_mask = seg_mask[:, crop_x:crop_x+h]
disparity = disparity[:, crop_x:crop_x+h]

# 6. Resize to 84x84
seg_mask = cv2.resize(seg_mask, (84, 84), interpolation=cv2.INTER_NEAREST)
disparity = cv2.resize(disparity, (84, 84), interpolation=cv2.INTER_LINEAR)

# 7. Stack into observation
obs = np.stack([seg_mask, disparity], axis=0)  # (2, 84, 84)

# 8. Frame stacking (maintain buffer of last 3 frames)
frame_buffer.append(obs)
stacked_obs = np.concatenate(frame_buffer[-3:], axis=0)  # (6, 84, 84)

# 9. Normalize for policy (float32, 0-1 range)
stacked_obs = stacked_obs.astype(np.float32) / 255.0
```

## Action Space

Policy outputs 6D continuous actions:

| Index | Joint | Range |
|-------|-------|-------|
| 0 | shoulder_pan | [-1, 1] |
| 1 | shoulder_lift | [-1, 1] |
| 2 | elbow | [-1, 1] |
| 3 | wrist_flex | [-1, 1] |
| 4 | wrist_roll | [-1, 1] |
| 5 | gripper | [-1, 1] |

Actions are delta positions scaled by action_repeat=2 in training.

## Loading the Policy

```python
import torch

# Load checkpoint
ckpt = torch.load("path/to/1200000_snapshot.pt")
encoder_state = ckpt["encoder"]
actor_state = ckpt["actor"]

# The encoder expects (batch, 6, 84, 84) input
# Output is actor's action distribution
```

## Key Files in pick-101/

| File | Purpose |
|------|---------|
| `src/training/train_efficientvit_seg.py` | Segmentation model definition |
| `src/training/infer_efficientvit_seg.py` | Segmentation inference utilities |
| `src/training/so101_factory.py` | SegDepthWrapper reference implementation |
| `configs/image_based/drqv2_lift_seg_depth_v19.yaml` | Training config |

## Notes

1. **Disparity normalization**: Depth Anything V2 outputs relative disparity. Normalize per-frame to 0-255.
2. **Frame stacking**: Initialize buffer with 3 copies of first frame.
3. **Action repeat**: In sim, actions were repeated 2x. May need tuning for real robot.
4. **Wrist lock**: Training used `lock_wrist: false` - wrist joints are active.

## Segmentation Inference Example

```python
from src.training.infer_efficientvit_seg import SegmentationInference

seg_model = SegmentationInference(
    checkpoint_path="outputs/efficientvit_seg/best.ckpt",
    device="cuda"
)

# Returns (H, W) mask with class IDs 0-5
mask = seg_model.predict(bgr_image)
```
