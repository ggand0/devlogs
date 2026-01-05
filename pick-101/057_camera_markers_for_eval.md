# Devlog 057: Camera Markers for Eval Videos

## Overview

Added `--show-cam-markers` flag to `eval_checkpoint.py` for visualizing wrist camera position and look direction in eval videos.

## Implementation

Uses dynamic scene geometry (same approach as `scripts/visualize_camera.py`) to add markers at runtime:

- **Red sphere** at camera world position
- **Green sphere** at view target (5cm along look direction)
- **Green cylinder** connecting them

The markers are added via `mujoco.mjv_initGeom()` after `update_scene()` but before `render()`, so they appear in external views (topdown, side, front, iso) but not in the wrist_cam view itself.

### Key Function

```python
def add_camera_markers(scene, cam_world_pos: np.ndarray, cam_dir: np.ndarray):
    """Add 3D markers to scene showing camera position and view direction."""
    # Target point 5cm along view direction
    view_target = cam_world_pos + cam_dir * 0.05

    # Red sphere at camera position
    mujoco.mjv_initGeom(scene.geoms[scene.ngeom], ...)

    # Green sphere at view target
    mujoco.mjv_initGeom(scene.geoms[scene.ngeom], ...)

    # Green cylinder connecting them
    mujoco.mjv_initGeom(scene.geoms[scene.ngeom], ...)
```

### Usage

```bash
MUJOCO_GL=egl uv run python src/training/eval_checkpoint.py \
    path/to/snapshot.pt \
    --show-cam-markers \
    --output_dir path/to/output
```

## Why Dynamic Markers vs XML Sites

Initially tried adding `<site>` elements to the XML with `group="4"` and enabling via `MjvOption.sitegroup[4]=1`. But:

1. XML sites are in gripper-local frame, need manual quaternion calculation for look direction
2. Dynamic approach computes world position from `data.cam_xpos` and direction from `data.cam_xmat` automatically
3. Matches existing implementation in `scripts/visualize_camera.py`

## Files Changed

- `src/training/eval_checkpoint.py` - Added `add_camera_markers()` function and `--show-cam-markers` flag

## Example Output

Video with markers: `runs/image_rl/20260104_103333/eval_cam_markers.mp4`
