# Devlog 017: Table Mesh Collision Visual Mismatch

**Date**: 2025-01-11
**Status**: KNOWN ISSUE - Needs proper fix

## Problem

The imported table mesh (`assets/meshes/wood_table.glb`) has a visual-collision mismatch in Genesis:
- **Numerically**: Cube sits at z=0.015m (correct for 3cm cube on z=0 surface)
- **Visually**: Cube appears to float ~12mm above the rendered table mesh

This was "fixed" by adjusting `TABLE_POS` z from `-0.73` to `-0.742`, pushing the mesh 12mm lower so its collision surface aligns with z=0. But this is a superficial fix - the visual mesh and collision mesh are still misaligned.

### Root Cause

Genesis generates collision geometry from the mesh, but:
1. The mesh may not be watertight
2. Genesis's convex decomposition may create collision hull that doesn't match visual surface
3. The GLB mesh origin may not be at the table surface

### Current Workaround

```python
# scene_config.py
TABLE_POS = (0.25, 0.0, -0.742)  # Was -0.73, adjusted for collision offset
```

### Proper Fix Options

1. **Use Box for table collision** (like test_finger_pad_tuning.py does):
   ```python
   gs.morphs.Box(size=(1.0, 0.7, 0.73), pos=(0.25, 0.0, -0.365), fixed=True)
   ```

2. **Fix the mesh origin** in Blender so surface is at local z=0

3. **Use separate visual and collision meshes** for the table

---

## Context: Collision Fixes Applied (from Devlog 016)

### MuJoCo Menagerie-Style Finger Collision

Implemented finger collision pads following [Genesis Issue #614](https://github.com/Genesis-Embodied-AI/Genesis/issues/614):

#### 1. Disabled Visual Mesh Collision
```xml
<default class="visual">
  <geom type="mesh" contype="0" conaffinity="0" group="2"/>
</default>
```

#### 2. Finger Collision Class
```xml
<default class="finger_collision">
  <geom type="box" contype="1" conaffinity="1" group="3"
        friction="2 0.01 0.001"
        solref="0.01 1"
        solimp="0.98 0.99 0.001"/>
</default>
```

#### 3. Box Pads (4 per finger)

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

#### 4. Gripper Close Limit
Changed from `-0.15` to `0.02` radians to avoid over-penetration (cube popping out).

### IK Controller Gripper Mapping
```python
# src/controllers/ik_controller.py
GRIPPER_CLOSED = 0.02  # Minimum close to avoid finger pad over-penetration
GRIPPER_OPEN = 0.5     # Fully open
# Linear mapping: action [-1, 1] → joint [CLOSED, OPEN]
gripper_val = (gripper_action + 1) / 2 * (GRIPPER_OPEN - GRIPPER_CLOSED) + GRIPPER_CLOSED
```

### Training Environment Changes
- **Robot-only MJCF**: `so101_robot_only.xml` (no cube in MJCF)
- **Separate cube entity**: `gs.morphs.Box` for proper inter-entity collision
- **Self-collision enabled**: `enable_self_collision=True`

---

## Test Results

| Test | Result | Cube Z |
|------|--------|--------|
| test_finger_pad_tuning.py (Box table) | SUCCESS | 0.0917m |
| verify_training_env.py (Mesh table) | SUCCESS | 0.0916m |

---

## Files Modified

- `models/so101/so101_new_calib.xml` - Finger collision pads, visual mesh collision disabled
- `models/so101/so101_robot_only.xml` - Robot-only MJCF for training
- `src/controllers/ik_controller.py` - Gripper action mapping
- `src/envs/lift_cube_env.py` - Robot-only MJCF, separate cube entity
- `src/envs/scene_config.py` - TABLE_POS z adjustment
- `tests/test_finger_pad_tuning.py` - Box table, collision visualization
- `tests/verify_training_env.py` - Training env grasp test

---

## Downloaded Assets (MuJoCo Menagerie)

Convex collision meshes from `trs_so_arm100` (currently disabled, need coordinate transformation):
- `models/so101/assets/Fixed_Jaw_Collision_1.stl`
- `models/so101/assets/Fixed_Jaw_Collision_2.stl`
- `models/so101/assets/Moving_Jaw_Collision_1.stl`
- `models/so101/assets/Moving_Jaw_Collision_2.stl`
- `models/so101/assets/Moving_Jaw_Collision_3.stl`

---

## Handoff Summary

### What Works
1. Gripper grasping with box collision pads - cube lifts to ~9cm
2. Training env (LiftCubeEnv) passes grasp test
3. Gripper action mapping prevents over-penetration

### Known Issues
1. **Table mesh visual-collision mismatch** - Cube appears floating visually but is numerically correct
2. **MuJoCo Menagerie convex meshes disabled** - Need coordinate transformation to align with SO-101 finger frames

### Next Steps
1. Fix table mesh collision properly (use Box or fix mesh origin)
2. Enable MuJoCo Menagerie convex collision meshes with proper transforms
3. Run RL training with updated collision geometry

### Key References
- [Genesis Issue #614](https://github.com/Genesis-Embodied-AI/Genesis/issues/614) - Gripper collision fix
- [Genesis Issue #2158](https://github.com/Genesis-Embodied-AI/Genesis/issues/2158) - `enable_mujoco_compatibility` flag
- [MuJoCo Menagerie trs_so_arm100](https://github.com/google-deepmind/mujoco_menagerie/tree/main/trs_so_arm100) - Reference collision geometry
