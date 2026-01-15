# Wood Table Setup with Proper Height

## Summary

Added a realistic wooden table from ManiSkill assets with proper 73cm height and wood texture. Fixed z-fighting issues between floor and table surfaces.

## Changes

### Table Setup

- Imported `table.glb` from ManiSkill's TableSceneBuilder assets
- Table has embedded wood texture that renders correctly in Genesis
- Scaled table height to 73cm (real desk height) using Z scale factor of 1.388
- Rotated 90° around Z axis so long edge (1.38m) runs along X axis

### Z-Fighting Fix

The original setup had z-fighting between:
1. MJCF floor plane at z=0
2. Table top surface at z=0

**Solution:**
- Removed floor plane from `models/so101/lift_cube.xml`
- Added Genesis floor plane at z=-0.73 (73cm below table top)
- Table mesh provides collision surface for cube at z=0

### Floor and Table Positioning

```python
# Floor at 73cm below table top
scene.add_entity(
    gs.morphs.Plane(pos=(0, 0, -0.73)),
    surface=gs.surfaces.Default(color=(0.3, 0.3, 0.32, 1.0)),
)

# Table mesh scaled to 73cm height
scene.add_entity(
    gs.morphs.Mesh(
        file=table_mesh_path,
        scale=(1.0, 1.0, 1.388),  # Scale height to 73cm
        pos=(0.25, 0.0, -0.73),   # Bottom at floor, top at z=0
        euler=(0, 0, 90),
        fixed=True,
        collision=True,
    ),
)
```

### Camera Views

Both wrist camera and wide camera now show:
- Wood grain texture on table surface
- No z-fighting artifacts
- Table legs visible in wide view
- Floor visible below table

## Files Modified

- `models/so101/lift_cube.xml` - Removed MJCF floor plane
- `tests/test_splitview_pick.py` - Added table mesh and floor setup

## Assets Added

- `assets/meshes/table.glb` - ManiSkill wood table mesh (from TableSceneBuilder)

## Robot Repositioning & Shadow Fix

### Shadow Disable for Camera Markers

Genesis's shadow rendering system (in `genesis/ext/pyrender/jit_render.py`) skips shadow casting for objects marked as transparent (`render_flags[5]`). Setting alpha < 1.0 triggers transparency detection.

**Solution:** Set marker colors to alpha=0.99 instead of 1.0:

```python
cam_sphere = scene.add_entity(
    gs.morphs.Sphere(...),
    surface=gs.surfaces.Default(color=(0.0, 1.0, 1.0, 0.99)),  # Cyan, no shadow
)

cam_arrow = scene.add_entity(
    gs.morphs.Cylinder(...),
    surface=gs.surfaces.Default(color=(1.0, 0.5, 0.0, 0.99)),  # Orange, no shadow
)
```

### Robot Position at Table Edge

Positioned robot at left edge of table (from wide camera view), rotated 90° CW to face inward.

**Table bounds** (measured via trimesh): Y range [-0.336, 0.354]

**Robot positioning:**
- Left edge at Y = 0.354
- Robot base radius ~4cm
- Robot center at Y = 0.354 - 0.04 = 0.314

```python
robot = scene.add_entity(gs.morphs.MJCF(
    file=str(model_path),
    pos=(0.25, 0.314, 0.0),
    euler=(0, 0, -90),  # -90° CW to face -Y (toward table center)
))
```

The MJCF morph `pos` and `euler` parameters transform the entire MJCF scene (robot + cube) at load time:
- Robot at (0.25, 0.314, 0) facing -Y
- Cube transformed to (0.25, 0.064, 0.027) - on the table, in front of rotated robot

### Grasp Offset Adjustment

After -90° Z rotation, coordinate axes changed:
- Original +X (forward) → now -Y
- Original +Y (side) → now +X

`finger_width_offset` (gripper centering) changed from Y to X axis:

```python
# Before rotation: Y offset
above_pos[1] += finger_width_offset

# After rotation: X offset
above_pos[0] += finger_width_offset
```

Applied to: `above_pos`, `grasp_target`, `lift_pos`

### Results

- Camera markers cast no shadows
- Robot positioned at left table edge, rear base aligned with edge
- Cube lift test passes (lift amount: 0.0568m > 0.03m threshold)

## Known Issues

None currently.

## Next Steps

- Match IRL camera angles and positions
