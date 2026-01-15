# Split-View Visualization with Wrist Camera

## Summary

Created a split-view visualization tool that shows the wrist camera view alongside a wide view with 3D camera markers. This helps debug and validate the wrist camera positioning and orientation.

## Split-View Layout

```
+------------------+----------------------+
|  Wrist Camera    |     Wide View        |
|    (480x480)     |     (640x480)        |
|                  |                      |
|  Square crop     |  Shows robot arm     |
|  from center     |  with 3D markers     |
+------------------+----------------------+
        Total: 1120x480
```

## Wrist Camera Configuration

Using exact MuJoCo wrist_cam parameters from `so101_new_calib.xml`:

```python
# Position in gripper local frame
pos = (0.02, -0.08, -0.02)  # x=right, y=behind fingers, z=toward wrist

# Quaternion: 180° yaw + -25° pitch
quat = (0, 0, -0.2164, 0.9763)

# FOV: 86° vertical (matches real innoMaker 130° camera at 4:3)
fov = 86

# Near plane reduced to prevent clipping
near = 0.01
```

### Quaternion to Rotation Matrix

```python
w, x, y, z = 0.0, 0.0, -0.2164, 0.9763

R_cam = [
    [1 - 2*(y*y + z*z), 2*(x*y - w*z),     2*(x*z + w*y)],
    [2*(x*y + w*z),     1 - 2*(x*x + z*z), 2*(y*z - w*x)],
    [2*(x*z - w*y),     2*(y*z + w*x),     1 - 2*(x*x + y*y)]
]
```

## 3D Camera Markers

Two markers visualize the wrist camera in the wide view:

1. **Cyan Sphere** (radius=0.015): Camera position
2. **Orange Cylinder** (radius=0.006, height=0.08): Look direction

### Marker Properties
- `fixed=True`: Not affected by physics simulation
- `collision=False`: No collision interference with robot/objects

### Look Direction Calculation

```python
# Camera looks along -Z in its local frame
# Transform through camera rotation, then gripper rotation
cam_look_local = R_cam @ [0, 0, -1]  # -Z in camera frame -> gripper frame
look_dir_world = R_gripper @ cam_look_local  # gripper frame -> world frame
```

## Center Crop

The wrist camera renders at 640x480 and is center-cropped to 480x480:

```python
h, w = wrist_img.shape[:2]  # 480, 640
crop_size = min(h, w)  # 480
x_start = (w - crop_size) // 2  # 80
wrist_cropped = wrist_img[:, x_start:x_start + crop_size]  # 480x480
```

## Usage

```bash
uv run python tests/test_splitview_pick.py
```

Output: `outputs/splitview_pick.mp4`

## Lighting Configuration

Genesis uses `VisOptions` to configure scene lighting:

```python
vis_options = gs.options.VisOptions(
    shadow=True,
    background_color=(0.2, 0.25, 0.3),
    ambient_light=(0.4, 0.4, 0.4),
    lights=[
        {'type': 'directional', 'dir': (-1, -1, -1), 'color': (1.0, 1.0, 1.0), 'intensity': 2.5},
        {'type': 'directional', 'dir': (1, 0.5, -1), 'color': (0.7, 0.8, 1.0), 'intensity': 1.0},
    ],
)
```

Note: RayTracer renderer (with point lights) requires LuisaRenderer which may not be available. The Rasterizer uses directional lights configured through VisOptions.

## Environment Setup

Matching the real-world setup:

```python
# Wooden table (visual only, no collision interference)
table = scene.add_entity(
    gs.morphs.Box(
        size=(0.8, 0.6, 0.75),  # 80x60cm, 75cm height
        pos=(0.2, 0.0, -0.375),  # Top surface at z=0
        fixed=True,
        collision=False,
    ),
    surface=gs.surfaces.Default(color=(0.55, 0.35, 0.2, 1.0)),  # Wood brown
)
```

## Physics Settings

Genesis defaults match MuJoCo:
- **Gravity**: -9.81 m/s² (Earth gravity)
- **Cube**: 3x3x3cm, 30g mass, wood-like friction (0.5)
- **Timestep**: 0.002s with 2 substeps

## Known Issues

1. **Camera orientation**: The view is close to the real camera but may need fine-tuning. The MuJoCo quaternion conversion might have subtle differences with Genesis camera conventions.

2. **Clipping**: Reduced with `near=0.01` but some objects very close to camera may still clip.

3. **Marker accuracy**: The 3D marker direction visualization is approximate - the cylinder orientation quaternion calculation may have edge cases.

4. **Lighting appearance**: The Genesis Rasterizer produces a somewhat flat/bright look compared to MuJoCo. Background and floor color settings may not fully take effect.

## Files

- `tests/test_splitview_pick.py`: Split-view visualization test
- `outputs/splitview_pick.mp4`: Output video

## Next Steps

1. Compare Genesis wrist camera output with real camera / MuJoCo rendering
2. Fine-tune camera rotation if needed
3. Use this visualization to debug RL training camera observations
