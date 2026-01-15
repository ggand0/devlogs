# Genesis Collision Mesh Options Research

Research on Genesis collision mesh handling from GitHub issues #614 and #2158.

## GitHub Issue #614: CoACD in URDF for Robotic Arm

**Problem:** SO-ARM-100 gripper jaw was "squishing the rigid ball like a marshmallow" during grasping.

**Key finding:** User disabled `convexify` option to get accurate collision mesh matching the visual jaw shape, but penetration still occurred.

**Quote from OP:**
> "Currently the collision mesh is accurately displaying the visual shape of the jaw links, after I disabled convexify option. Compare Video 5 against Video 4."

**Resolution:** Issue closed via PR #950 after switching to MJCF with pre-computed convex decomposition. The SO-ARM-100 MJCF model uses multiple convex shapes to approximate jaw geometry.

**Takeaway:** Convex decomposition of non-convex collision meshes improves gripper stability.

## GitHub Issue #2158: SO101 and Cube Physics Problem

Same gripper (SO-101), same grasping problem.

### Suggestions from maintainer (duburcqa):

**1. Simulation timing:**
> "Is this video real-time? I don't think so. You should set max_FPS based on the simulation timestep: max_FPS=1/dt."

**2. Remove custom options first:**
> "I would first try to remove all custom options and see what you get. constraint_timeconst=0.01 may be too large. I would also avoid defining custom material parameters for now."

**3. MuJoCo compatibility mode:**
```python
rigid_options=gs.options.RigidOptions(
    enable_mujoco_compatibility=True,
)
```

**4. Reduce constraint time constant:**
> "Penetration is too large. To avoid this, you must reduce the time constant."

User tried `constraint_timeconst=0.005` but concluded:
> "I think the so101 grabber design is not very good. If they swap it to three finger clamp design would have been better. This is not the physics simulator platform's issue."

**5. Final resolution:**
> "After updating the collision geometry, it is able to pickup the cube."

The fix was redesigning the collision geometry - described as going from "scissors" to proper gripper shape.

## MJCF Morph Options for Collision Control

From Genesis API (`gs.morphs.MJCF`):

| Option | Default | Description |
|--------|---------|-------------|
| `convexify` | True (rigid) | Convert meshes to convex hulls |
| `decompose_nonconvex` | deprecated | Use convexify + thresholds instead |
| `decompose_object_error_threshold` | 0.15 | Skip decomposition if volume diff < 15% |
| `decompose_robot_error_threshold` | inf | Skip decomposition for robots by default |
| `coacd_options` | None | Fine-tune CoACD convex decomposition |
| `decimate` | True | Simplify mesh |
| `decimate_face_num` | 500 | Target face count |

### Key options to try:

```python
# Option 1: Disable convexify to use original mesh shape
gs.morphs.MJCF(
    file=str(model_path),
    convexify=False,
)

# Option 2: Force convex decomposition with tighter threshold
gs.morphs.MJCF(
    file=str(model_path),
    convexify=True,
    decompose_robot_error_threshold=0.0,  # Force decomposition
)

# Option 3: Custom CoACD options
gs.morphs.MJCF(
    file=str(model_path),
    coacd_options=gs.options.CoacdOptions(
        threshold=0.05,  # Tighter approximation
        max_convex_hull=32,
    ),
)
```

## Editing Collision Meshes in Blender

For custom collision geometry:

1. **Export STL from MJCF mesh references:**
   ```
   models/so101/wrist_roll_follower_so101_v1.stl  (static finger)
   models/so101/moving_jaw_so101_v1.stl          (moving finger)
   ```

2. **Create collision-specific mesh in Blender:**
   - Open STL: `blender models/so101/wrist_roll_follower_so101_v1.stl`
   - Simplify: Decimate modifier or manual editing
   - Convex hull: Edit Mode → Select All → Mesh → Convex Hull
   - Or create box primitives covering contact areas
   - Export as `wrist_roll_follower_so101_v1_collision.stl`

3. **Update MJCF to use separate collision mesh:**
   ```xml
   <!-- Visual mesh (full detail) -->
   <geom type="mesh" class="visual" mesh="wrist_roll_follower_so101_v1"/>
   <!-- Collision mesh (simplified) -->
   <geom type="mesh" class="collision" mesh="wrist_roll_follower_so101_v1_collision"/>
   ```

4. **Add mesh asset:**
   ```xml
   <asset>
     <mesh file="wrist_roll_follower_so101_v1_collision.stl"/>
   </asset>
   ```

## Options Tested So Far

| Option | Result |
|--------|--------|
| `use_gjk_collision=True` | Helps with primitives, not enough for mesh |
| `enable_mujoco_compatibility=True` | Works, similar to without |
| Disable mesh collision (`contype="0"`) | Works but leaves floating boxes |
| Box finger pads only | Works but visually wrong |

## Next Steps

1. Try `convexify=False` on MJCF to get accurate mesh collision
2. Try tighter `decompose_robot_error_threshold=0.0` to force better decomposition
3. Create custom collision STLs in Blender if needed
4. Consider hybrid: mesh collision for arm, box pads for fingertips only
