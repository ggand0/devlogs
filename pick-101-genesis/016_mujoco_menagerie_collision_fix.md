# Devlog 016: MuJoCo Menagerie Collision Fix for Genesis Grasping

**Date**: 2025-01-11
**Status**: SUCCESS - Cube lifted to 0.0917m

## Problem

Genesis simulator had issues with gripper collision causing:
1. Cube being pushed through the table during gripper close
2. Cube "popping off" or ejecting when collision pads penetrated too deeply
3. Unstable grasping with mesh collision enabled

## Solution

Following the approach from [Genesis Issue #614](https://github.com/Genesis-Embodied-AI/Genesis/issues/614), we implemented MuJoCo Menagerie-style collision geometry.

### Key Changes

#### 1. Disable Visual Mesh Collision

Changed `class="visual"` default from `contype="1"` to `contype="0"`:

```xml
<default class="visual">
  <geom type="mesh" contype="0" conaffinity="0" group="2"/>
</default>
```

This prevents visual meshes from participating in collision - only dedicated collision geometry handles contacts.

#### 2. Add Finger Collision Class

Created a new default class for finger collision pads with tuned contact properties:

```xml
<default class="finger_collision">
  <geom type="box" contype="1" conaffinity="1" group="3"
        friction="2 0.01 0.001"
        solref="0.01 1"
        solimp="0.98 0.99 0.001"/>
</default>
```

#### 3. MuJoCo Menagerie-Style Finger Pads

Added 4 collision pads per finger along the inner surface (matching MuJoCo Menagerie `trs_so_arm100`):

**Static Finger (Fixed Jaw):**
```xml
<geom name="fixed_jaw_pad_1" class="finger_collision" size="0.003 0.005 0.006" pos="-0.0089 0 -0.1014"/>
<geom name="fixed_jaw_pad_2" class="finger_collision" size="0.003 0.005 0.006" pos="-0.0099 0 -0.0914"/>
<geom name="fixed_jaw_pad_3" class="finger_collision" size="0.003 0.006 0.010" pos="-0.0106 0 -0.0768"/>
<geom name="fixed_jaw_pad_4" class="finger_collision" size="0.003 0.006 0.012" pos="-0.0113 0 -0.0572"/>
```

**Moving Finger (Moving Jaw):**
```xml
<geom name="moving_jaw_pad_1" class="finger_collision" size="0.001 0.005 0.004" pos="-0.0113 -0.077 0.019"/>
<geom name="moving_jaw_pad_2" class="finger_collision" size="0.001 0.005 0.004" pos="-0.0093 -0.067 0.019"/>
<geom name="moving_jaw_pad_3" class="finger_collision" size="0.001 0.008 0.005" pos="-0.0073 -0.055 0.019"/>
<geom name="moving_jaw_pad_4" class="finger_collision" size="0.001 0.010 0.008" pos="-0.0073 -0.035 0.019"/>
```

#### 4. Less Aggressive Gripper Close

Changed gripper closed position from `-0.15` to `0.02` radians to avoid over-penetration:

```python
# Original MuJoCo uses action space [-1, 1] mapped to ctrlrange
# action -0.8 → ctrl ≈ 0.017 rad (nearly closed, not fully)
GRIPPER_CLOSED = 0.02  # Instead of -0.15
```

## Downloaded Assets

Downloaded convex collision meshes from MuJoCo Menagerie `trs_so_arm100`:
- `Fixed_Jaw_Collision_1.stl`
- `Fixed_Jaw_Collision_2.stl`
- `Moving_Jaw_Collision_1.stl`
- `Moving_Jaw_Collision_2.stl`
- `Moving_Jaw_Collision_3.stl`

These are currently disabled (commented out) due to coordinate transformation issues, but available for future use.

## Results

| Metric | Before | After |
|--------|--------|-------|
| Cube Z during close | Drops to 0.004m | Stays at 0.015m |
| Final cube Z | 0.015m (failed) | 0.0917m (success) |
| Contact type | Mesh (unstable) | Box pads (stable) |
| Contacts at close | 6-21 (inconsistent) | 12 (consistent) |

## Files Modified

- `models/so101/so101_new_calib.xml` - Main collision geometry changes
- `tests/test_finger_pad_tuning.py` - Gripper close value
- `tests/view_collision_geometry.py` - Collision visualization tool

## References

- [Genesis Issue #614](https://github.com/Genesis-Embodied-AI/Genesis/issues/614) - Original gripper collision fix
- [Genesis Issue #2158](https://github.com/Genesis-Embodied-AI/Genesis/issues/2158) - `enable_mujoco_compatibility` flag
- [MuJoCo Menagerie trs_so_arm100](https://github.com/google-deepmind/mujoco_menagerie/tree/main/trs_so_arm100) - Reference collision geometry

## Additional Changes (Training Environment)

### LiftCubeEnv Updates

- **Robot-only MJCF**: Changed from `lift_cube.xml` to `so101_robot_only.xml`
- **Separate cube entity**: Cube is now a separate `gs.morphs.Box` entity for proper inter-entity collision
- **Self-collision enabled**: Added `enable_self_collision=True` to rigid_options

### IK Controller Updates

- **Configurable DOF indices**: Added `arm_dof_indices` and `gripper_dof_index` parameters
- **Robot-only support**: Works with both full MJCF (12 DOFs) and robot-only (6 DOFs)
- **Gripper action mapping**: Added proper mapping in `compute_joint_targets()`:
  ```python
  # Map gripper action from [-1, 1] to gripper joint position
  GRIPPER_CLOSED = 0.02  # Minimum close to avoid finger pad over-penetration
  GRIPPER_OPEN = 0.5     # Fully open
  # Linear mapping: action [-1, 1] → joint [CLOSED, OPEN]
  gripper_val = (gripper_action + 1) / 2 * (GRIPPER_OPEN - GRIPPER_CLOSED) + GRIPPER_CLOSED
  ```

### Scene Config Updates

- **Table position adjusted**: `TABLE_POS` z changed from `-0.73` to `-0.742` for mesh collision offset

### Test Updates

- **verify_training_env.py**: Enhanced with collision mode visualization, more detailed logging

### README Updates

- **Debug Tools section**: Added collision geometry viewer command

## Next Steps

1. Run training with updated collision geometry
2. Consider enabling convex collision meshes with proper coordinate transformation

## Verification

Training env grasp test passed:
- `tests/verify_training_env.py` - Cube lifted to 0.0916m (SUCCESS)
- `tests/env_grasp_test.mp4` - Video showing successful grasp with training env
