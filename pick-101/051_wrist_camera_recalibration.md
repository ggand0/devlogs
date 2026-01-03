# Wrist Camera Recalibration (v4)

**Date:** 2025-01-03

## Problem

The wrist camera in simulation showed skybox in the lower portion of the frame, while the real innoMaker camera only sees the ground plane in the same grasped pose. This indicated the camera was tilted too far backward/upward.

## Root Causes Identified

### 1. Incorrect Pitch Sign
Previous calibration (v3) had pitch = +40° in the XML (`euler="0.698 0 3.14159"`), but comments said -40°. The positive pitch was tilting the camera upward, showing skybox.

### 2. Wrong FOV Value
Was using `fovy=103°` which is the **horizontal** FOV of the real camera. But MuJoCo's `fovy` is **vertical** FOV.

For 4:3 aspect ratio (640x480):
- Horizontal FOV: 103°
- Vertical FOV: ~86° (calculated from aspect ratio)
- Diagonal FOV: 130°

Using 103° as vertical FOV gave ~17° more vertical coverage than reality, causing the camera to see too much of the gripper body.

## Final Calibration (v4)

| Parameter | v3 (old) | v4 (new) | Notes |
|-----------|----------|----------|-------|
| Position X | 0.02 | 0.02 | Centers between fingers |
| Position Y | -0.08 | -0.08 | Forward toward fingertips |
| Position Z | -0.06 | -0.02 | Moved toward wrist |
| Pitch | +40° | -25° | Tilts forward, not backward |
| Yaw | 180° | 180° | Unchanged |
| Roll | 0° | 0° | Unchanged |
| FOV | 103° | 86° | Corrected to vertical FOV |

### XML Configuration
```xml
<camera name="wrist_cam" pos="0.02 -0.08 -0.02" euler="-0.436 0 3.14159" fovy="86"/>
```

Note: euler pitch in radians: -25° = -0.436 rad

## Testing Process

Generated 50+ test videos in `runs/camera_test_v4/` varying:
- Pitch angles: -10° to -80°
- Z positions: -0.06 to +0.06
- Y positions: -0.04 to -0.10
- FOV values: 86°, 103°, 120°
- X positions: 0.00 to 0.04

Key test videos:
- `042_pitch_30neg_z_neg0p02_y_neg0p08_fov86.mp4` - FOV fix validation
- `043_pitch_25neg_z_neg0p02_y_neg0p08_fov86.mp4` - Final settings
- `044_pitch_25neg_fov86_640x480_crop.mp4` - 4:3 → 1:1 crop validation

## Real Camera Setup

- **Camera:** innoMaker USB (130° diagonal FOV)
- **Resolution:** 640x480 (4:3)
- **Processing:** Center crop to 480x480, resize to 84x84

## Remaining Issues

1. **X position bias:** Camera may be slightly biased toward one finger. Current x=0.02 is approximate - may need fine-tuning after mounting real camera.

2. **Aspect ratio mismatch:** Sim renders 84x84 (1:1) directly, real camera is 640x480 (4:3) then cropped. This means sim sees less horizontal FOV than cropped real image.

## Files Changed

- `models/so101/so101_new_calib.xml` - Updated camera settings
- `scripts/visualize_camera.py` - Updated default calibration

## Commit

`ae5e83d` - Recalibrate wrist camera to match real innoMaker 640x480
