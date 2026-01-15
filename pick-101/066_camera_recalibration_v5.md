# Devlog 066: Camera Recalibration v5

## IRL Camera Geometry Measurements

Measured from the real SO-101 robot with innoMaker wrist camera:

```
x - gripperframe (TCP)
|
|
A = 9cm (vertical from gripperframe to finger neck)
|
|
+----B-----*
     6.5cm (horizontal)

+: center of finger at gripper neck
*: camera position
```

### Position Calculation

```python
# Gripperframe is at z = -0.0981 in gripper local frame
# Finger neck is A=9cm above gripperframe
finger_neck_z = -0.0981 + 0.09 = -0.0081 ≈ -0.008

# Camera Y is B=6.5cm horizontal from finger neck
camera_y = -0.065

# Camera position
cam_pos = [0.01, -0.065, -0.008]
```

### Pitch Calculation

Camera looks at gripperframe + forward offset along -Z axis (toward fingertips):

```python
# NOTE: "forward" is along -Z axis (toward fingertips), NOT -Y axis
target = gripperframe + [0, 0, -forward]  # forward cm in -Z direction
target = [0, 0, gripperframe_z - forward]

# Best pitch with yaw=180° convention
# Use scripts/calculate_camera_params.py to compute
```

### FOV Calculation

innoMaker USB camera specs:
- Diagonal FOV: 130°
- Horizontal FOV: 103°
- Native resolution: 1080p (1920x1080, 16:9)

**Problem: Specs are inconsistent**

The stated diagonal (130°) and horizontal (103°) FOVs cannot both be correct for a 16:9 sensor:

```python
# Method 1: VFOV from HFOV and 16:9 aspect ratio
tan(VFOV/2) = tan(103°/2) × (9/16) = 1.257 × 0.5625 = 0.707
VFOV = 70.5°

# Verify diagonal from these values
DFOV = 2 × atan(√(tan²(51.5°) + tan²(35.3°))) = 110.5°  # NOT 130°!

# Method 2: VFOV from diagonal formula
tan²(VFOV/2) = tan²(130°/2) - tan²(103°/2) = 4.60 - 1.58 = 3.02
VFOV = 120.2°

# But this implies aspect ratio 0.724:1 (portrait!), not 16:9
```

**Solution: Trust HFOV=103° as ground truth**

Pipeline: 16:9 native → 4:3 mode (640x480) → 1:1 square crop (480x480)

```python
# Focal length from native HFOV
f = 1920 / (2 × tan(51.5°)) = 763.6 pixels

# Step 1: Native 16:9 (1920x1080)
HFOV = 103°, VFOV = 70.5°

# Step 2: Crop to 4:3 (1440x1080) - keep full height, crop width
HFOV = 2 × atan(1440 / (2 × 763.6)) = 86.6°
VFOV = 70.5° (unchanged)

# Step 3: Crop to 1:1 (1080x1080) - crop width to match height
HFOV = VFOV = 70.5°
```

**Result: MuJoCo fovy = 70.5°** (not 86° as previously calculated)

See `scripts/calculate_fov.py` for full calculation.

## Final Settings (v5)

| Parameter | v5 | v5 | Notes |
|-----------|-----|-----|-------|
| Position X | 0.01 | **0.008** | Fine-tuned |
| Position Y | -0.07 | **-0.065** | B=6.5cm from gripper center |
| Position Z | -0.015 | **-0.019** | Fine-tuned |
| Pitch | -23.2° | **-22.8°** | Fine-tuned |
| Yaw | 180° | 180° | Unchanged |
| FOV | 70.5° | **70.5°** | Unchanged |

### XML Format

```xml
<camera name="wrist_cam" pos="0.008 -0.065 -0.019" euler="0.398 0 3.14159" fovy="70.5"/>
```

Note: euler pitch in radians: +22.8° = +0.398 rad

**IMPORTANT: MuJoCo euler sign convention**

MuJoCo XML euler angles have opposite sign from scipy's `R.from_euler('xyz', ...)`.
The `visualize_camera.py` script uses scipy which requires `-22.8°`, but the XML requires `+22.8°` (+0.398 rad) to achieve the same orientation.

Verification:
```python
# scipy (visualize_camera.py)
rot = R.from_euler('xyz', [-22.8, 0, 180], degrees=True)
q = rot.as_quat()  # gives [0, 0, -0.197, 0.980]

# MuJoCo XML euler="0.398 0 3.14159"
# produces quat [0, 0, +0.197, 0.980]  # same rotation!
```

## Visualization Script Changes

Updated `apply_calibrated_camera()` in `scripts/visualize_camera.py`:
- Position: `[0.008, -0.065, -0.019]`
- Euler: `[-22.8, 0, 180]` degrees
- FOV: `70.5°`

## Test Output

- `runs/camera_test_v5/combined_grid.png` - 4-view comparison at 512x512
- `runs/camera_test_v5/camera_demo.mp4` - Motion test video
