# Devlog 048: Sim-to-Real Image Normalization Mismatch

**Date**: 2025-01-19
**Status**: Fixed

## Problem

DrQ-v2 pretrained on Genesis simulation produces broken outputs on real robot due to image normalization mismatch. The vision encoder receives values ~255x smaller than expected.

## Root Cause

### Expected Pipeline (RoboBase/Genesis)
1. Genesis outputs images as uint8 [0, 255]
2. Encoder with `normalise_inputs=True` does: `x / 255.0 - 0.5`
3. Result: float32 [-0.5, 0.5]

### Actual Pipeline (LeRobot Real Robot)
1. Camera outputs images as uint8 [0, 255]
2. `preprocess_observation()` does: `img /= 255` → [0, 1]
3. Encoder with `normalise_inputs=True` does: `x / 255.0 - 0.5`
4. Result: float32 **[-0.5, -0.496]** (double normalization!)

### Impact
- CNN encoder receives values ~255x smaller than training
- Feature extraction produces near-zero or garbage outputs
- Policy cannot perceive the environment correctly

## Code Evidence

### RoboBase encoder (`external/robobase/robobase/models/encoder.py:103-104`)
```python
if self._normalise_inputs:
    x = x / 255.0 - 0.5
```

### LeRobot preprocess (`lerobot/src/lerobot/envs/utils.py:68-70`)
```python
img = img.type(torch.float32)
img /= 255  # <-- Already normalized to [0, 1]
```

### LeRobot DrQ-v2 encoder (`lerobot/src/lerobot/policies/drqv2/modeling_drqv2.py:238-239`)
```python
if self._normalise_inputs:
    x = x / 255.0 - 0.5  # <-- Normalizes AGAIN!
```

## Solution

Add `normalize_images` config option to control image normalization:

### 1. `lerobot/src/lerobot/envs/utils.py`
```python
def preprocess_observation(
    observations: dict[str, np.ndarray],
    normalize_images: bool = True,  # New parameter
) -> dict[str, Tensor]:
    ...
    if normalize_images:
        img /= 255  # normalize to [0, 1]
    # else: keep as [0, 255] for encoders with normalise_inputs=True
```

### 2. `lerobot/src/lerobot/envs/configs.py`
```python
@dataclass
class EnvTransformConfig:
    ...
    normalize_images: bool = True  # If False, keep images as float32 [0, 255]
```

### 3. `lerobot/src/lerobot/scripts/rl/gym_manipulator.py`
```python
class ConvertToLeRobotObservation(gym.ObservationWrapper):
    def __init__(self, env, device: str = "cpu", normalize_images: bool = True):
        self.normalize_images = normalize_images
        ...

    def observation(self, observation):
        observation = preprocess_observation(observation, normalize_images=self.normalize_images)
```

### 4. `train_config.json`
```json
"wrapper": {
    ...
    "normalize_images": false  // For DrQ-v2 with normalise_encoder_inputs=True
}
```

### 5. Offline Buffer (Dataset Images)

The offline dataset also normalizes images to [0,1] during video decoding:
```python
# In video_utils.py line 165 and 240:
closest_frames = closest_frames.type(torch.float32) / 255
```

This causes a mismatch between live environment (now [0,255]) and offline buffer ([0,1]).

**Fix in `lerobot/src/lerobot/utils/buffer.py`**:
```python
def from_lerobot_dataset(
    ...
    unnormalize_images: bool = False,  # New parameter
) -> "ReplayBuffer":
    ...
    # Unnormalize images from [0, 1] to [0, 255]
    if unnormalize_images and ".images." in key and val.ndim == 3:
        val = val * 255.0
```

**Fix in `lerobot/src/lerobot/scripts/rl/learner.py`**:
```python
# When normalize_images=False in live env, we need to unnormalize dataset images
normalize_images = getattr(cfg.env.wrapper, "normalize_images", True)
unnormalize_images = not normalize_images
...
offline_replay_buffer = ReplayBuffer.from_lerobot_dataset(
    ...
    unnormalize_images=unnormalize_images,
)
```

**IMPORTANT**: Delete cached offline buffer after this fix:
```bash
rm outputs/hilserl_drqv2/offline_buffer.pt
```

## Other Potential Issues Identified

### 1. Gripper Action Space Mismatch (MEDIUM)
- Genesis: continuous [-1, 1]
- Real robot: quantized to discrete {0, 1, 2} via `gripper_quantization_threshold: 0.8`
- Impact: Fine gripper control lost

### 2. Workspace Bounds Mismatch (LOW)
| Axis | Genesis | Real Robot |
|------|---------|-----------|
| X | [0.05, 0.45] | [0.15, 0.35] |
| Y | [-0.2, 0.35] | [-0.15, 0.15] |
| Z | [0.01, 0.4] | [0.02, 0.25] |
- Impact: Policy may attempt out-of-bounds actions (clipped)

### 3. ImageResizeWrapper Clamp (FIXED)
`ImageResizeWrapper` was clamping to [0, 1] which conflicted with `normalize_images=False`.
**Fixed by removing the clamp entirely** - this was unnecessary and now supports both [0, 1] and [0, 255] image ranges.

## Files Modified

1. `lerobot/src/lerobot/envs/utils.py` - Added `normalize_images` parameter
2. `lerobot/src/lerobot/envs/configs.py` - Added `normalize_images` field to EnvTransformConfig
3. `lerobot/src/lerobot/scripts/rl/gym_manipulator.py` - Updated ConvertToLeRobotObservation and make_robot_env
4. `lerobot/src/lerobot/utils/buffer.py` - Added `unnormalize_images` parameter to from_lerobot_dataset()
5. `lerobot/src/lerobot/scripts/rl/learner.py` - Pass unnormalize_images to offline buffer creation

## Verification

After fix, verify image values at encoder input:
```python
# Should see values in [0, 255] range before encoder normalization
print(f"Image range: [{obs['observation.images.gripper_cam'].min()}, {obs['observation.images.gripper_cam'].max()}]")
```
