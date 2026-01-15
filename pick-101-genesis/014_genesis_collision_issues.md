# Genesis Collision Issues Investigation

Continuation of devlog 013. This documents additional collision issues discovered while debugging finger pad grasping.

## Issue 1: Table Mesh Visual/Collision Mismatch

### Problem
Cube appeared to float ~12mm above the table surface visually, even though collision detection showed it resting on a surface.

### Root Cause
The table mesh (`assets/meshes/table.glb`) is **not watertight**:
```
[CoACD] [info] Manifold Check
[CoACD] [info]   Unclosed mesh
[CoACD] [info] Mesh Manifoldness: false
```

Genesis uses CoACD (Convex Approximate Decomposition) to generate collision geometry from non-convex meshes. When the mesh is not manifold/watertight, CoACD's convex hull approximation adds padding, resulting in:

- Visual table top: Z = 0.5255
- Collision table top: Z = 0.538
- **Difference: 12.6mm**

### Verification
```python
# Cube dropped from height onto unscaled table at origin
Cube settled at Z: 0.5531
Cube bottom at Z: 0.5381
Mesh visual bounds: Z from 0.0000 to 0.5255
Difference (collision - visual): 12.6mm
```

### Solution
Replace mesh table with Box primitive for accurate collision:
```python
# Box table with top at z=0
scene.add_entity(
    gs.morphs.Box(
        size=(1.0, 0.7, 0.73),  # x, y, z dimensions
        pos=(0.25, 0.0, -0.365),  # Center so top is at z=0
        fixed=True,
    ),
    surface=gs.surfaces.Default(color=(0.4, 0.3, 0.2, 1.0)),
    material=gs.materials.Rigid(friction=0.5),
)
```

After fix, cube settles at Z = 0.0150 (correct for 3cm cube on table at z=0).

## Issue 2: Gripper Mesh Penetration

### Problem
During gripper close, the finger meshes **penetrate through the cube** instead of pushing against it. This causes:
1. Cube gets pushed downward (into the table)
2. Cube Z drops from 0.015 to 0.005 during gripper close
3. No stable pinch grip is achieved
4. Cube slips during lift

### Observations from test output
```
Phase 3: Closing gripper...
  Step 0: gripper=0.500, cube_z=0.0150
  Step 30: gripper=0.459, cube_z=0.0149
  Step 60: gripper=0.364, cube_z=0.0147
  Step 90: gripper=0.240, cube_z=0.0111  <- Cube pushed down
  Step 120: gripper=0.109, cube_z=0.0057  <- Cube pushed through table
```

The cube Z drops from 0.015 to 0.006 during close - it's being pushed down ~9mm into/through the table surface.

### Contact Analysis
```
Cube contacts after close: 20
  Contact: ext:1 <-> ext:34 (table-cube)
  Contact: ext:34 <-> gripper:MESH
  Contact: gripper:BOX <-> ext:34
  Contact: ext:34 <-> moving_jaw_so101_v1:MESH
```

Both BOX and MESH geoms are making contact, but the mesh collision is dominating and causing penetration rather than clean pushing.

### Possible Causes
1. **Insufficient collision iterations**: Genesis constraint solver may not be resolving interpenetration fast enough
2. **Mesh collision resolution**: SDF-based mesh collision may have lower resolution than box primitives
3. **Gripper velocity too high**: Fingers closing too fast for collision detection
4. **Missing collision groups**: Mesh geoms may need different contype/conaffinity settings

## Issue 3: Finger Pad Box Position

### Current Configuration (2x size, 5mm cubes)
From MJCF (`so101_new_calib.xml`):
```xml
<!-- Static finger pad -->
<geom name="static_finger_pad" type="box"
      size="0.0025 0.0025 0.0025"
      pos="-0.010125 0.0 -0.100"
      friction="1 0.05 0.001"/>

<!-- Moving finger pad -->
<geom name="moving_finger_pad" type="box"
      size="0.0025 0.0025 0.0025"
      pos="-0.01011 -0.076 0.019"
      friction="1 0.05 0.001"/>
```

### World Positions During Grasp (gripper closed)
```
gripper box:           [0.225, 0.006, 0.013]
moving_jaw_so101_v1:   [0.254, 0.003, 0.013]
Cube center:           [0.250, 0.000, 0.005]  <- pushed down
```

The box pads are at similar Z height (0.013) but the cube has been pushed down to Z=0.005.

## Related Issue

[Genesis Issue #2158](https://github.com/Genesis-Embodied-AI/Genesis/issues/2158) - SO101 and Cube Physics Problem

Same gripper, same problem. Key findings:
- **`use_gjk_collision=True`** improved but didn't fully resolve
- **Root cause**: Gripper collision geometry was "like trying to grab the cube with a pair of scissors"
- **Solution**: Redesign collision geometry for the gripper

## Solution: Disable Mesh Collision on Fingertips

### Why This Works

The issue is that mesh collision geometry in Genesis uses SDF (Signed Distance Field) representation which:
1. Has lower contact point resolution than primitive shapes
2. Can cause interpenetration when meshes close on an object
3. Creates unstable contact forces that push the cube sideways/downward

By disabling mesh collision on the finger parts and relying solely on the BOX pad geoms:
1. Contact points are well-defined and stable (box-box contact)
2. No interpenetration - boxes push cleanly against the cube
3. Friction forces apply correctly for stable grasping

### Scene Configuration Requirements

For stable grasping, the scene needs:
```python
gs.options.RigidOptions(
    constraint_solver=gs.constraint_solver.Newton,
    enable_collision=True,
    enable_joint_limit=True,
    enable_self_collision=True,
    use_gjk_collision=True,  # Better collision for primitives
)
```

And a Box table (not mesh) to avoid the 12mm collision offset:
```python
scene.add_entity(
    gs.morphs.Box(
        size=(1.0, 0.7, 0.73),
        pos=(0.25, 0.0, -0.365),  # Top at z=0
        fixed=True,
    ),
    ...
)
```

### MJCF Changes

In `models/so101/so101_new_calib.xml`, added `contype="0" conaffinity="0"` to the finger mesh geoms to disable their collision. Only the BOX pad geoms remain for contact.

**gripper body** (static finger) - 2 mesh geoms modified:
```xml
<!-- sts3215_03a_v1_5 servo housing at fingertip -->
<geom type="mesh" class="visual" contype="0" conaffinity="0"
      pos="0.0077 0.0001 -0.0234"
      quat="0.707107 -0.707107 1.66015e-15 6.45094e-15"
      mesh="sts3215_03a_v1" material="sts3215_03a_v1_material"/>

<!-- wrist_roll_follower_so101_v1 finger structure -->
<geom type="mesh" class="visual" contype="0" conaffinity="0"
      pos="8.32667e-17 -0.000218214 0.000949706"
      quat="0 1 0 0"
      mesh="wrist_roll_follower_so101_v1" material="wrist_roll_follower_so101_v1_material"/>
```

**moving_jaw_so101_v1 body** (moving finger) - 1 mesh geom modified:
```xml
<!-- moving_jaw_so101_v1 finger mesh -->
<geom type="mesh" class="visual" contype="0" conaffinity="0"
      pos="-5.55112e-17 -5.55112e-17 0.0189"
      quat="1 -0 3.00524e-16 -2.00834e-17"
      mesh="moving_jaw_so101_v1" material="moving_jaw_so101_v1_material"/>
```

The finger pad BOX geoms remain unchanged with `contype="1"` (default collision enabled):
```xml
<!-- static_finger_pad on gripper body -->
<geom name="static_finger_pad" type="box" size="0.0025 0.0025 0.0025"
      pos="-0.010125 0.0 -0.100" friction="1 0.05 0.001" rgba="0 0 1 1"/>

<!-- moving_finger_pad on moving_jaw_so101_v1 body -->
<geom name="moving_finger_pad" type="box" size="0.0025 0.0025 0.0025"
      pos="-0.01011 -0.076 0.019" friction="1 0.05 0.001" rgba="0 1 0 1"/>
```

### Results

**Before** (mesh collision enabled):
```
Cube contacts: gripper:MESH, moving_jaw:MESH (penetration)
Cube Z during close: 0.015 -> 0.005 (pushed down 10mm into table)
Final cube Z: 0.015m - FAIL (slipped)
```

**After** (mesh collision disabled):
```
Cube contacts: gripper:BOX, moving_jaw:BOX (only box pads)
Cube Z during close: 0.015 -> 0.015 (stable)
Final cube Z: 0.093m - SUCCESS (lifted 8cm!)
```

### Notes

- Fingers still visually penetrate the cube (the mesh geoms pass through)
- But only the BOX pads participate in physics collision
- This gives stable contact points for grasping
- The 5mm box pads are sufficient with mesh collision disabled

## Test Scripts

- `tests/test_finger_pad_tuning.py` - Standalone scene with closeup camera, Box table
- `tests/test_grasp_stability.py` - Uses LiftCubeEnv

## Video Output

`tests/finger_pad_tuning.mp4` shows:
- Closeup view of fingers during grasp
- Cube being pushed down as fingers close (before fix)
- Successful lift with only BOX contacts (after fix)

## TODO: Apply to LiftCubeEnv

The training environment `src/envs/lift_cube_env.py` still uses:
1. Mesh table (`assets/meshes/table.glb`) - needs Box primitive
2. Missing `use_gjk_collision=True` in RigidOptions

The MJCF changes in `so101_new_calib.xml` are shared, so finger collision is already fixed. But the table and scene options need updating for consistent behavior between test scripts and training.
