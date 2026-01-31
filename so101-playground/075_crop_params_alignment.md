# Crop Parameters Alignment Across Training Pipeline

## Issue

Crop parameters were inconsistent across different parts of the training pipeline:

1. **HIL-SERL Training**: Used `[0, 80, 480, 480]` crop
2. **Reward Classifier Preview**: Used manual center crop calculation
3. **Reward Classifier Training**: **No crop at all** - images squished from 640x480 to 128x128

This caused the reward classifier to be trained on distorted images but inference received properly cropped images.

## Crop Format

The crop format is `(top, left, height, width)` - used by `torchvision.transforms.functional.crop()`:

```python
# gym_manipulator.py:992-993
crop_params_dict: Dictionary mapping image observation keys to crop parameters
                  (top, left, height, width).

# Usage at line 1047
obs[k] = F.crop(obs[k], *self.crop_params_dict[k])
```

For horizontal center crop of 640x480 → 480x480:
```json
"crop_params_dict": {
    "observation.images.gripper_cam": [0, 80, 480, 480]
}
```
- top=0: start from row 0
- left=80: (640-480)/2 = 80, center horizontally
- height=480: full height
- width=480: square output

## Fixes

### 1. Reward Classifier Live Preview
```python
# scripts/reward_classifier_live_preview.py
# BEFORE: Dynamic center crop calculation
h, w = frame.shape[:2]
crop_x = (w - 480) // 2
cropped = frame[:, crop_x:crop_x+480]

# AFTER: Explicit params matching training config
crop_top, crop_left, crop_h, crop_w = 0, 80, 480, 480
cropped = frame[crop_top:crop_top+crop_h, crop_left:crop_left+crop_w]
```

### 2. Reward Classifier Training Script
Added `apply_crop()` function and config support:
```python
# scripts/train_reward_classifier.py
def apply_crop(batch: dict, crop_params: dict, image_keys: list[str]) -> dict:
    for key in image_keys:
        if key in batch and key in crop_params:
            top, left, h, w = crop_params[key]
            batch[key] = TF.crop(batch[key], top, left, h, w)
    return batch
```

### 3. Reward Classifier Training Config
```json
// configs/reward_classifier_grasponly_v4_train_config.json
"dataset": {
    "crop_params_dict": {
        "observation.images.gripper_cam": [0, 80, 480, 480]
    },
    ...
}
```

## Result

All three components now use identical crop: `[0, 80, 480, 480]`
- HIL-SERL training (gym_manipulator ImageCropResizeWrapper)
- Reward classifier live preview
- Reward classifier training

## Note

The reward classifier needs to be **retrained** with the correct crop params for the fix to take effect.
