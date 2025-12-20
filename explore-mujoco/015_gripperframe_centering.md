# Devlog 015: Gripperframe Y-Centering Fix

## Summary

Fixed a subtle Y-axis bias in the `gripperframe` site that caused the IK target to be offset toward the static finger instead of centered between both fingers.

## The Issue

In `so101_new_calib.xml`, the gripperframe site had:
```xml
<site group="3" name="gripperframe" pos="-0.0079 -0.000218121 -0.0981274" .../>
```

The `Y=-0.0079` component biased the site toward the static finger (left side when viewed from front). This meant when IK targeted a position, the fingers would be slightly offset from the intended grasp point.

## The Fix

Centered the gripperframe by zeroing the X and Y offsets:
```xml
<site group="3" name="gripperframe" pos="0.0 0.0 -0.0981274" .../>
```

The Z component (`-0.0981274`) remains unchanged - this projects the TCP ~10cm forward along the gripper's local -Z axis, which transforms to the fingertip region in world coordinates when the wrist is in top-down grasp pose.

## Visualization Debugging

Created `visualize_sites.py` to render mocap spheres at site positions:
- RED sphere: gripperframe (TCP)
- GREEN sphere: graspframe
- BLUE sphere: finger midpoint (computed from geom positions)

Key learnings:
- Use `emission="1"` material property for marker visibility
- Place temp XML in same directory as includes to resolve paths
- Mocap bodies with `contype="0" conaffinity="0"` for non-colliding markers

## Clarification from Previous Session

The session summary incorrectly stated "Wrong IK end-effector site (gripperframe vs graspframe)". This was **incorrect**:

- **gripperframe**: At fingertips when wrist points down - CORRECT for IK
- **graspframe**: ~6cm behind fingertips - WRONG for IK (as documented in devlog 011)

The only actual issue was the Y-axis centering, not the choice of site.

## Confirmed Real Issues (from previous session)

1. **Wrist locking during step()**: Added `lock_wrist` parameter to `step()` that forces `wrist_flex=1.65` and `wrist_roll=π/2` for stable top-down grasping

2. **Gripper oscillation**: When `lock_wrist=True` and agent commands close, the gripper action is locked to the reset value instead of following raw agent actions

## Non-Issues

- **Action scale**: Not a bug, just a tuning parameter. Current config uses `action_scale=0.3` which works well for gradual movements

## Files Changed

- `SO-ARM100/Simulation/SO101/so101_new_calib.xml` - Centered gripperframe Y position
- `visualize_sites.py` - New debugging script for site visualization
