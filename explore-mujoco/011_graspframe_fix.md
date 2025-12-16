# Devlog 011: The Graspframe Fix - Why V1-V7 Could Never Grasp

## The Root Cause

After extensive debugging of why the agent could never achieve `(True, True)` contact (both gripper sides touching the cube), we discovered a fundamental issue: **the reach reward was measuring distance from the wrong point**.

## The Problem: gripperframe vs Actual Fingers

The SO-101 model has a site called `gripperframe` defined in `so101_new_calib.xml`:

```xml
<site group="3" name="gripperframe" pos="-0.0079 -0.000218121 -0.0981274" .../>
```

This site is positioned **~10cm in front of the gripper body** (along the local -Z axis). It's the "Tool Center Point" (TCP) - a standard robotics convention for teleoperation and motion planning.

The environment was using this point for:
1. IK targeting (via `ik.get_ee_position()`)
2. The `gripper_pos` sensor
3. The reach reward calculation: `gripper_to_cube = norm(gripper_pos - cube_pos)`

**The actual finger geoms are 7cm behind this point.**

### Measured Positions

```
gripperframe site (TCP):     [0.391, 0.000, 0.226]
Static finger geom 27:       [0.321, 0.000, 0.224]
Moving jaw geom 29:          [0.341, -0.001, 0.253]
Finger midpoint:             [0.331, -0.000, 0.236]

Distance from TCP to finger midpoint: 7.0cm
```

## The Claw Machine Analogy

Imagine teaching someone to use a claw machine. You say: **"Get close to the prize!"** and reward them based on distance to the prize.

But you're measuring from a point 10cm in front of the claw fingers:

```
         Measurement point (gripperframe/TCP)
                   |
                   v
    [====|====]----X
     fingers
         ^
    Where grasping actually happens (7cm behind X)
```

When the agent minimizes distance and gets "X" on top of the cube, it thinks it's winning - but the actual fingers are 7cm away, unable to grasp.

**This explains V7's behavior perfectly**: The agent touched the cube with the gripper palm/base (one-sided contact) because that's the closest gripper part to the measurement point. Moving the fingers to the cube would *increase* the measured distance, reducing the reward.

## The Fix

### 1. Added `graspframe` site (between fingers)

In `SO-ARM100/Simulation/SO101/so101_new_calib.xml`:

```xml
<!-- Frame gripperframe (TCP - tool center point, 10cm in front) -->
<site group="3" name="gripperframe" pos="-0.0079 -0.000218121 -0.0981274" .../>
<!-- Frame graspframe (midpoint between fingers, for RL reach reward) -->
<site group="3" name="graspframe" pos="0.002338 0.000055 -0.037843" .../>
```

### 2. Added `grasp_pos` sensor

In `SO-ARM100/Simulation/SO101/lift_cube_scene.xml`:

```xml
<sensor>
    <framepos name="cube_pos" objtype="body" objname="cube" />
    <framepos name="gripper_pos" objtype="site" objname="gripperframe" />
    <framepos name="grasp_pos" objtype="site" objname="graspframe" />
</sensor>
```

### 3. Updated reach reward calculation

In `envs/lift_cube.py`:

```python
def _get_info(self) -> dict[str, Any]:
    gripper_pos = self.ik.get_ee_position()
    cube_pos = self.data.sensor("cube_pos").data.copy()
    grasp_pos = self.data.sensor("grasp_pos").data.copy()

    # Use grasp_pos (between fingers) for reach reward, not gripper_pos (TCP)
    gripper_to_cube = np.linalg.norm(grasp_pos - cube_pos)
```

## Before vs After

```
OLD measurement point               NEW measurement point

         gripperframe                     graspframe
              |                               |
              v                               v
[====|====]---X                     [====|====]
  fingers                             fingers
                                          ^
                                   Now measuring here!
```

**OLD**: Reach reward maximized when TCP is at cube (fingers 7cm away, can't grasp)
**NEW**: Reach reward maximized when fingers are at cube (can actually grasp)

![Comparison of measurement points](011_graspframe_comparison.png)

## Verification

```python
# After fix
gripper_pos (TCP):     [0.391, 0.000, 0.226]
grasp_pos (new):       [0.331, -0.000, 0.237]
finger midpoint:       [0.331, -0.000, 0.239]

Distance grasp_pos to finger midpoint: 0.22cm  # Correct!
```

## Why This Wasn't Caught Earlier

1. **V1 "worked" via physics exploit** - Soft contacts (`solref="0.02 1"`) let the gripper clip through the cube, triggering both contact flags without real grasping
2. **The bug was in the reward, not the physics** - After fixing to stiff contacts, we assumed the problem was reward shaping, not the measurement point
3. **TCP is standard** - Using `gripperframe` as the reference point is correct for teleoperation; it just shouldn't be used for the reach reward

## Impact

This fix means:
- V7 reward structure might actually work now (no changes to reward logic needed)
- All previous training runs (V1-V7 with stiff physics) were fundamentally handicapped
- The agent was being rewarded for the wrong behavior

## Next Steps

1. Train V7 with the graspframe fix
2. If successful, this validates that the reward structure was fine - only the measurement point was wrong
3. May not need staged rewards or HER after all

## Files Changed

- `SO-ARM100/Simulation/SO101/so101_new_calib.xml` - Added `graspframe` site
- `SO-ARM100/Simulation/SO101/lift_cube_scene.xml` - Added `grasp_pos` sensor
- `envs/lift_cube.py` - Use `grasp_pos` for reach reward calculation
