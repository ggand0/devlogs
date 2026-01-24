# 068: ImageCropResizeWrapper Clamp Bug Fix

## Bug

The reward classifier worked in standalone preview but failed during HIL-SERL training.

**Root cause**: `ImageCropResizeWrapper` in `gym_manipulator.py` always clamped images to `[0, 1]`:

```python
obs[k] = obs[k].clamp(0.0, 1.0)  # BUG
```

When `normalize_images=False` (used for DrQ-v2 compatibility), images from `ConvertToLeRobotObservation` are `float32 [0, 255]`. Clamping to `[0, 1]` destroyed all pixel data - every value > 1 became 1.0, resulting in a nearly white image.

## Data Flow

```
RobotEnv (uint8 HWC [0,255])
    ↓
ConvertToLeRobotObservation (float32 CHW, [0,255] when normalize_images=False)
    ↓
ImageCropResizeWrapper (BUGGY clamp to [0,1] destroyed data)
    ↓
RewardWrapper (received garbage image, classifier failed)
```

## Fix

Added `normalize_images` parameter to `ImageCropResizeWrapper`:

```python
self.clamp_max = 1.0 if normalize_images else 255.0
obs[k] = obs[k].clamp(0.0, self.clamp_max)
```

Updated `make_robot_env` to pass the flag:

```python
env = ImageCropResizeWrapper(
    env=env,
    crop_params_dict=cfg.wrapper.crop_params_dict,
    resize_size=cfg.wrapper.resize_size,
    normalize_images=normalize_images,  # NEW
)
```

## Additional Changes

- Added `display_reward_preview` config option to show reward overlay during training
- Lowered reward threshold from 0.85 to 0.7
- Added `display_reward_preview` field to `EnvTransformConfig` dataclass

## Lesson

Always verify data ranges at each step of the preprocessing pipeline. The standalone preview and training environment used different code paths with different normalization assumptions.
