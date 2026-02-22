# Devlog 069: Distance-Based Reward Shaping

**Date**: 2025-01-26
**Status**: Potential Approach (Not Implemented)

## Problem

The reward classifier only provides positive reward when the gripper is near the cube (for grasping). This creates a **sparse reward problem**:

- No learning signal for "move toward cube"
- Policy has no gradient to discover reaching behavior
- Random exploration unlikely to stumble into reward zone
- After 100+ episodes, policy still can't reach cube autonomously

## Proposed Solution: Distance-Based Reward Shaping

Add intermediate reward proportional to gripper-cube distance:

```python
# Instead of just:
reward = 10.0 if success else -0.05

# Add distance shaping:
distance = np.linalg.norm(gripper_pos - cube_pos)
distance_reward = -0.1 * distance  # Closer = less penalty
reward = 10.0 if success else (-0.05 + distance_reward)
```

## Challenge: Cube Position Detection

Need to compute cube 3D position automatically.

### For Table-Constrained Tasks (Cube Always on Table)

**Ray-plane intersection** with known table height:

1. Segment cube → centroid `(u, v)` in pixels
2. Back-project to camera ray: `ray_cam = K_inv @ [u, v, 1]`
3. Transform ray to world frame using gripper pose from FK
4. Intersect ray with table plane `z = table_height`
5. Intersection point = cube 3D position

```python
def pixel_to_world(u, v, camera_intrinsics, gripper_pose, table_height):
    # Back-project to camera ray
    K_inv = np.linalg.inv(camera_intrinsics)
    ray_cam = K_inv @ np.array([u, v, 1.0])
    ray_cam = ray_cam / np.linalg.norm(ray_cam)

    # Transform to world frame
    R, t = gripper_pose[:3, :3], gripper_pose[:3, 3]
    ray_world = R @ ray_cam
    origin = t

    # Intersect with table plane (z = table_height)
    t_intersect = (table_height - origin[2]) / ray_world[2]
    point_3d = origin + t_intersect * ray_world

    return point_3d
```

**Requirements:**
- Camera intrinsics matrix K (from checkerboard calibration)
- Hand-eye calibration (camera-to-gripper transform)
- Table height measurement

### For General Cube Positions (Any Height)

Without plane constraint, single RGB + segmentation gives no depth. Options:

1. **Known object size** (most practical):
   ```python
   depth = (focal_length * real_cube_size) / pixel_cube_size
   ```
   - Measure apparent cube size from segmentation mask
   - Known real cube dimensions → estimate depth

2. **External depth camera** - Fixed position looking at workspace

3. **Monocular depth estimation** - ML models (DepthAnything, MiDaS) but less accurate

4. **Multi-view triangulation** - Move gripper, take two images, triangulate

**Note**: No depth cameras small enough to mount on SO101 gripper.

### Simplest Option: 2D Pixel Distance

For gripper-mounted camera, as gripper approaches cube:
- Cube centroid moves toward image center
- Distance = pixels from cube centroid to image center
- No calibration needed, fast to compute

```python
def compute_2d_distance_reward(segmentation_mask, image_shape):
    # Find centroid of segmented cube
    coords = np.argwhere(segmentation_mask > 0)
    if len(coords) == 0:
        return 0.0  # No cube detected
    centroid = coords.mean(axis=0)

    # Distance from image center
    center = np.array([image_shape[0] / 2, image_shape[1] / 2])
    pixel_distance = np.linalg.norm(centroid - center)

    # Normalize by image diagonal
    max_dist = np.linalg.norm(center)
    normalized_distance = pixel_distance / max_dist

    return -0.1 * normalized_distance  # Reward for being centered
```

## Error Sources

- Segmentation centroid ≠ true cube center
- Camera calibration quality
- FK accuracy
- Table not perfectly flat
- Lighting variations affecting segmentation

For reward shaping, ~1-2cm accuracy is acceptable. Not doing precision grasping, just learning "am I getting closer?"

## Prerequisites

Already have:
- Segmentation model + annotations
- FK for gripper position
- MuJoCo model

Need:
- Camera intrinsics calibration
- Table height measurement
- Integration into RewardWrapper

## Alternative: BC Pretraining

Instead of solving the sparse reward with distance shaping, could pretrain SAC actor with behavioral cloning on demonstration data. This teaches reaching behavior through imitation, bypassing the sparse reward problem entirely.

See `scripts/bc_pretrain_sac.py` for implementation.

## Decision

TBD - Either implement distance-based reward shaping or use BC pretraining approach.
