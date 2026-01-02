# Devlog 043: Camera FOV Calibration

## Real Camera Specs (innoMaker USB Camera)

From Amazon product page:
- **Diagonal FOV**: 130°
- **Horizontal FOV**: 103°
- **Resolution**: 1080p @ 30fps
- **Interface**: USB 2.0 UVC

## FOV Calculations

### Vertical FOV Derivation

Using the relationship between diagonal, horizontal, and vertical FOV:
```
tan²(diag/2) = tan²(h/2) + tan²(v/2)
```

With diagonal = 130° and horizontal = 103°:
```
tan²(65°) = tan²(51.5°) + tan²(v/2)
4.60 = 1.58 + tan²(v/2)
tan²(v/2) = 3.02
tan(v/2) = 1.74
v/2 = 60.1°
v ≈ 120°
```

**Vertical FOV ≈ 120°**

### Effective FOV for 1:1 Square Crop

For RL training, we use 84×84 images (1:1 aspect ratio). When center-cropping from the full sensor:

| Aspect Ratio | Limiting Dimension | Effective FOV |
|--------------|-------------------|---------------|
| 16:9 (native) | - | H: 103°, V: 120° |
| 1:1 (square crop) | Horizontal | **103°** |

**For MuJoCo `fovy` parameter: use 103°** (not 130°)

### Previous Incorrect Setting

The original sim camera used `fovy=75`, which was significantly narrower than the real camera's effective 103° FOV for square crops.

## MuJoCo Camera Configuration

### Current Settings (in XML)
```xml
<camera name="wrist_cam" pos="0.0 -0.055 0.02" euler="0 0 3.14159" fovy="75"/>
```

### Recommended Settings
```xml
<camera name="wrist_cam" pos="0.0 -0.055 0.02" euler="[TBD]" fovy="103"/>
```

## Final Calibrated Settings

After systematic testing of position and angle parameters:

| Parameter | Value | Notes |
|-----------|-------|-------|
| **Position X** | 0.02 | Centers fingers in camera view |
| **Position Y** | -0.08 | Forward toward fingertips |
| **Position Z** | -0.06 | Down along gripper body |
| **Pitch** | -40° | Tilts camera to view workspace |
| **Yaw** | 180° | Camera faces forward |
| **Roll** | 0° | No roll |
| **FOV** | 103° | Matches real camera 1:1 crop |

### XML Configuration
```xml
<camera name="wrist_cam" pos="0.02 -0.08 -0.06" euler="-0.698 0 3.14159" fovy="103"/>
```

Note: Euler angles in radians: pitch = -40° = -0.698 rad, yaw = 180° = 3.14159 rad

### Verification
- Camera view shows both fingers centered
- Side view confirms camera positioned between fingers
- Top-down view shows camera pointing toward workspace
- Visualization: `runs/camera_test_v3/005_final_calibrated.png`

## Test Results

Camera position/angle tests saved to: `runs/camera_test_v3/`

| File | Description |
|------|-------------|
| `001_z_positions.png` | Z positions -0.06 to -0.09 (closer to fingertips) |
| `002_pitch_variations.png` | Pitch -50° to -80° at z=-0.08 |
| `003_y_positions.png` | Y positions -0.055 to -0.10 (forward/back) |
| `004_best_candidates.png` | Best candidate configurations |
| `005_final_calibrated.png` | **Final config with 4-column view (cam, iso, top, side)** |

## Notes

- MuJoCo cameras look along -Z in their local frame
- Euler angles use XYZ convention (pitch is X rotation)
- Negative pitch tilts camera downward toward fingertips
- Y position (more negative = further forward on gripper)
- Z position (more negative = closer to fingertips)
