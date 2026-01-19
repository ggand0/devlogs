# Locked Joint IK Reset Fix

## Problem

During HIL-SERL actor operation, the wrist joints (3=wrist_flex, 4=wrist_roll) were rotating to incorrect angles, despite recording working correctly with 90° wrist angles.

## Root Cause

**Missing calibration in train_config.json**

The train_config.json robot section had:
- `"id": null` - should be `"ggando_so101_follower"`
- `"calibration_dir": null` - should be the calibration path
- `"use_degrees": false` - should be `true`

Without proper calibration, the raw encoder values weren't being transformed correctly, so 90° in the code didn't match the physical 90° orientation.

## Fix

Updated `outputs/hilserl_drqv2/train_config.json` to match `record_config.json`:

```json
"robot": {
    "type": "so101_follower_end_effector",
    "id": "ggando_so101_follower",
    "calibration_dir": "/home/gota/.cache/huggingface/lerobot/calibration/robots/so101_follower",
    ...
    "use_degrees": true,
    ...
},
"teleop": {
    "type": "so101_leader",
    "id": "ggando_so101_leader",
    "calibration_dir": "/home/gota/.cache/huggingface/lerobot/calibration/teleoperators/so101_leader",
    ...
}
```

## Additional Fixes (not the root cause)

### draccus config parsing workaround

Changed `locked_joints` and `locked_joint_positions` to use `None` default with `__post_init__` to work around draccus not properly overriding `default_factory` values from JSON.

### Delta clamping re-enforcement

Added step 6a in gym_manipulator.py IK reset to re-enforce locked joint positions after delta clamping.

## Lesson Learned

When something works in one command but not another with the same config values, **compare the full configs first** before diving into code debugging.
