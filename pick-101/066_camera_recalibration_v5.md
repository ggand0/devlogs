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

Camera looks at gripperframe + 4cm forward (to center the workspace in view):

```python
target = gripperframe + [0, -0.04, 0]  # 4cm forward
target = [0, -0.04, -0.0981]

# Best pitch with yaw=180° convention
pitch = -15.5°
```

## Final Settings (v5)

| Parameter | v4 | v5 | Notes |
|-----------|-----|-----|-------|
| Position X | 0.02 | **0.01** | Closer to static finger |
| Position Y | -0.08 | **-0.065** | From IRL measurement (B=6.5cm) |
| Position Z | -0.02 | **-0.008** | From IRL measurement (A=9cm) |
| Pitch | -25° | **-15.5°** | Calculated to look at gripperframe+4cm |
| Yaw | 180° | 180° | Unchanged |
| FOV | 86° | 86° | Unchanged |

### XML Format

```xml
<camera name="wrist_cam" pos="0.01 -0.065 -0.008" euler="-0.2705 0 3.14159" fovy="86"/>
```

Note: euler pitch in radians: -15.5° = -0.2705 rad

## Visualization Script Changes

Updated `apply_calibrated_camera()` in `scripts/visualize_camera.py`:
- Position: `[0.01, -0.065, -0.008]`
- Euler: `[-15.5, 0, 180]` degrees

## Test Output

- `runs/camera_test_v5/combined_grid.png` - 4-view comparison at 512x512
- `runs/camera_test_v5/camera_demo.mp4` - Motion test video
