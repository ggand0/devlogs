# 070: Disparity Format for Sim-to-Real Depth

## Problem

MuJoCo depth and Depth Anything V2 use different representations. For sim-to-real transfer, they must match exactly.

## MuJoCo Depth Buffer

```python
renderer.enable_depth_rendering()
depth = renderer.render()  # [0, 1], 0=near, 1=far (linear depth)
```

## Depth Anything V2 Output

DA V2 outputs **affine-invariant inverse depth (disparity)**:
- Normalized to [0, 1]
- **1 = nearest pixel**
- **0 = farthest pixel**
- Formula: `disparity = 1 / depth`

Source: https://github.com/DepthAnything/Depth-Anything-V2

## Conversion

To match DA V2 format in sim:

```python
def depth_to_disparity(depth):
    """Convert MuJoCo linear depth to disparity (DA V2 format).

    Args:
        depth: MuJoCo depth [0, 1] where 0=near, 1=far

    Returns:
        disparity: [0, 1] where 1=near, 0=far (matching DA V2)
    """
    eps = 1e-3  # Avoid division by zero
    disparity = 1.0 / (depth + eps)

    # Normalize to [0, 1]
    d_min, d_max = disparity.min(), disparity.max()
    disparity_norm = (disparity - d_min) / (d_max - d_min)

    return disparity_norm
```

## Why This Matters

| Property | MuJoCo Linear | DA V2 Disparity |
|----------|---------------|-----------------|
| Near objects | Low values (0) | High values (1) |
| Far objects | High values (1) | Low values (0) |
| Scale | Linear | Inverse (1/d) |

If sim outputs linear depth but real uses disparity, the policy won't transfer because:
- Relative depth relationships are inverted
- Non-linear vs linear scaling affects gradients differently

## Test Video

Updated `tests/test_wrist_cam_seg_video.py` to output 3-column video:
- RGB | Disparity | Segmentation
- Disparity now uses proper `1/depth` conversion matching DA V2

Output: `outputs/wrist_cam_seg.mp4`

## SegDepthWrapper Implementation

Added `SegDepthWrapper` in `src/training/so101_factory.py` for 2-channel observation.

### Input Format

| Channel | Content | Range | Notes |
|---------|---------|-------|-------|
| 0 | Segmentation class IDs | 0-4 | Same 5-class scheme as SegmentationWrapper |
| 1 | Disparity | 0-255 | Normalized inverse depth matching DA V2 |

### Observation Shape

- Per frame: `(2, 84, 84)` uint8
- With frame_stack=3: `(6, 84, 84)`

### Config

```yaml
env:
  obs_type: seg_depth  # rgb, seg, or seg_depth
```

Config file: `configs/image_based/drqv2_lift_seg_depth_v19.yaml`

### Eval Video Visualization

Updated `MultiCameraVideoRecorder` in `workspace.py`:
- 3 columns: wrist_cam | depth | wide
- Middle column now shows disparity from wrist cam (replaces closeup angle)
- For seg_depth mode, wrist_cam column shows RGB | Disparity | Seg stacked

### Real Robot Pipeline

```
RGB Camera
    ├─→ Custom Seg Model (~5ms) → 5-class mask (channel 0)
    └─→ Depth Anything V2 Small (~10ms) → disparity (channel 1)
                    ↓
            Concat: (2, 84, 84)
                    ↓
              RL Policy
```
