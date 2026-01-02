# Finger Pad Collision Boxes for Stable Grasping

## Problem

During top-down pick, the cube wobbled and tilted when lifted. The moving finger would penetrate the cube slightly, causing instability even with `impratio=10` and `cone="elliptic"` settings.

## Root Cause

From [MuJoCo GitHub Issue #941](https://github.com/google-deepmind/mujoco/issues/941):

> Long collision geoms (like finger meshes) with contact occurring only at the tips cause oscillation. The single contact point from mesh collision cannot accurately capture the surface contact, leading to instabilities that manifest as sliding or wobbling.

The SO-101 gripper uses mesh collision geometry for the fingers. When grasping, contact occurs at the fingertips but the collision detection generates a single unstable contact point.

## Solution

Add small invisible box geoms at the fingertips as collision pads. This approach is used by the Franka Panda gripper model in MuJoCo Menagerie.

Box geoms provide:
- Stable multi-point contact surfaces
- Predictable collision behavior
- Better friction force distribution

## Implementation

Added two 2.5mm cube collision pads to `so101_new_calib.xml`:

### Static Finger Pad (on gripper body)
```xml
<geom name="static_finger_pad" type="box"
      size="0.00125 0.00125 0.00125"
      pos="-0.008875 0.0 -0.100"
      rgba="1 0.5 0.5 0.8"
      friction="1 0.05 0.001"/>
```

### Moving Finger Pad (on moving_jaw_so101_v1 body)
```xml
<geom name="moving_finger_pad" type="box"
      size="0.00125 0.00125 0.00125"
      pos="-0.01136 -0.076 0.019"
      rgba="0.5 0.5 1 0.8"
      friction="1 0.05 0.001"/>
```

### Position Tuning

The boxes are positioned to slightly protrude inward from the finger mesh surfaces:
- Too much protrusion: visible gap between finger and object
- Too little protrusion: box buried inside mesh, ineffective
- Sweet spot: ~0.1-0.2mm protrusion, barely visible but effective

For the static finger (gripper body frame):
- Local -Z points toward fingertip
- Local +X points inward toward grasp center
- More positive X = more protrusion inward

For the moving finger (moving_jaw body frame):
- Different local frame due to quaternion rotation
- More negative X = more protrusion inward (opposite of static)

### Contact Detection Update

Updated `test_topdown_pick.py` to detect contact with pad geoms:

```python
static_pad_geom_id = mujoco.mj_name2id(model, mujoco.mjtObj.mjOBJ_GEOM, "static_finger_pad")
moving_pad_geom_id = mujoco.mj_name2id(model, mujoco.mjtObj.mjOBJ_GEOM, "moving_finger_pad")

def is_grasping():
    contacts = get_contacts()
    has_static = static_pad_geom_id in contacts
    has_moving = moving_pad_geom_id in contacts
    return has_static and has_moving
```

## Results

Before: Cube wobbles and tilts during lift, unstable grasp
After: Stable lift with no wobble, cube holds steady at Z=0.09

```
3. Closing gripper...
   Contact at step 207, gripper=-0.66
   Grasping: True

4. Lifting...
   step 0: cube_z=0.016, grasping=True
   step 100: cube_z=0.050, grasping=True
   step 200: cube_z=0.088, grasping=True

5. Holding...
   step 0: cube_z=0.090, grasping=True
   step 100: cube_z=0.090, grasping=True

Final cube Z: 0.0906
Result: SUCCESS
```

## References

- [MuJoCo Issue #941 - Oscillation with long collision geoms](https://github.com/google-deepmind/mujoco/issues/941)
- [MuJoCo Menagerie - Franka Panda gripper](https://github.com/google-deepmind/mujoco_menagerie/tree/main/franka_emika_panda) - Uses box pads at fingertips
- [MuJoCo XML Reference - multiccd flag](https://mujoco.readthedocs.io/en/stable/XMLreference.html) - Alternative approach for mesh-mesh contact

## Related

- [devlogs/019_mujoco_contact_slip_fix.md](019_mujoco_contact_slip_fix.md) - impratio and elliptic cone settings
