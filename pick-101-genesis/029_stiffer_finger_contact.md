# Devlog 029: Stiffer Finger Contact for PLA/Wood

**Date:** 2026-01-16

## Problem

Eval videos from devlog 028 showed finger penetration into the cube during grasping. Despite 50% success rate improvement, the contact solver was too soft for PLA/wood materials.

## Root Cause

The `solimp` parameter in finger_collision class was set to `0.98 0.99 0.001`, copied from MuJoCo Menagerie's rubber gripper (trs_so_arm100). This allows significant penetration suitable for soft rubber but not for hard PLA plastic.

## MuJoCo solimp Parameter

`solimp = (dmin, dmax, width)` controls contact impedance:
- Higher values (closer to 1) = softer, more penetration allowed
- Lower values = stiffer, less penetration

| Setting | solimp | Material Type |
|---------|--------|---------------|
| Previous | 0.98 0.99 0.001 | Soft rubber |
| MuJoCo default | 0.9 0.95 0.001 | General purpose |
| Current | 0.9 0.95 0.001 | PLA/wood (hard) |

## Change Made

**File:** `models/so101/so101_new_calib.xml`

```xml
<!-- Before (soft rubber) -->
<default class="finger_collision">
  <geom ... solimp="0.98 0.99 0.001"/>
</default>

<!-- After (PLA/wood) -->
<default class="finger_collision">
  <geom ... solimp="0.9 0.95 0.001"/>
</default>
```

## Verification

IK grasp test: **100% success (10/10)**

Video: `outputs/ik_grasp_multipos.mp4`

## Commit

`cc0d283` - Stiffen finger contact for PLA/wood material

## Next Steps

1. Run training with stiffer contacts to see if success rate improves beyond 50%
2. Review video for reduced penetration
3. If still penetrating, try even stiffer values (0.85 0.9 0.001)

## References

- [MuJoCo XML Reference - solimp](https://mujoco.readthedocs.io/en/stable/XMLreference.html)
- [Devlog 028](028_physics_fix_training_results.md) - Previous training results showing penetration
