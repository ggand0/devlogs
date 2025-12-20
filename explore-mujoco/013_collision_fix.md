# Devlog 013: Collision System Fix

## Problem

After enabling collision for all gripper geoms, the arm stopped being able to reach target positions. The IK controller was commanding joint movements but the arm wasn't responding.

## Root Cause

When collision was enabled for all geoms (changing visual class default from `contype="0" conaffinity="0"` to `contype="1" conaffinity="1"`), the arm body parts started colliding with each other:

```
Initial contacts: 10
  geom_1 <-> geom_9   (base <-> shoulder)
  geom_1 <-> geom_10
  geom_2 <-> geom_9
  geom_2 <-> geom_10
  geom_3 <-> geom_9
  geom_3 <-> geom_10
```

The base and shoulder bodies were interpenetrating, creating contact forces that prevented the arm from moving.

## Solution

Added contact exclusions for adjacent bodies in `so101_new_calib.xml`:

```xml
<contact>
  <!-- Exclude self-collision between adjacent arm bodies -->
  <exclude body1="base" body2="shoulder"/>
  <exclude body1="shoulder" body2="upper_arm"/>
  <exclude body1="upper_arm" body2="lower_arm"/>
  <exclude body1="lower_arm" body2="wrist"/>
  <exclude body1="wrist" body2="gripper"/>
  <exclude body1="gripper" body2="moving_jaw_so101_v1"/>
</contact>
```

After fix, only expected contacts remain:
```
Contacts after exclusions: 4
  floor <-> cube_geom (x4)
```

## Other Changes

1. **Collision class default**: Changed from no explicit contype/conaffinity to `contype="1" conaffinity="1"` to be explicit

2. **Gripper force**: Increased from `-3.35 3.35` to `-10 10` for stronger grip

3. **Cube size**: Changed from 2x2x2cm to 3x3x3cm to fit gripper gap better

## Remaining Issue

The finger still hits the cube center instead of straddling it. The Y offset calculation needs adjustment to position the fingers on either side of the cube.
