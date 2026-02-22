# Devlog 047: Locked Joints Not Enforced During Teleoperation

**Date**: 2025-01-19
**Status**: Fixed

## Problem

The `locked_joints: [3, 4]` config is NOT enforced during leader-follower teleoperation. The recorded dataset has joints 3 and 4 varying significantly (60°-100°) instead of being locked at 90° as Genesis expects.

### Evidence

Dataset analysis of `gtgando/so101_pick_lift_cube_locked_wrist`:
```
Joint 3 & 4 values across 18 episodes:
 Ep | J3 mean | J3 std | J4 mean | J4 std
--------------------------------------------------
  0 |    78.0 |    6.5 |    84.4 |   10.4
  1 |    60.3 |   12.7 |    90.2 |    7.5
  2 |    82.5 |   17.1 |    92.6 |    7.8
  ...
Genesis expects: J3=90°, J4=90° (locked)
```

## Root Cause

The `locked_joints` config in `record_config.json` only affects:
1. IK-based resets (`_ik_reset_to_ee_target`)
2. IK-based end-effector control (`_ik_solve`)

It does NOT affect leader-follower teleoperation because:
- `_handle_intervention()` at line 1857 directly copies leader positions to follower
- `RobotEnv.step()` at line 459 sends these positions without modification

### Code Path

1. `_handle_intervention()` (line 1857):
   ```python
   self.unwrapped._leader_positions = {name: leader_pos_dict[name] for name in leader_pos_dict}
   ```

2. `RobotEnv.step()` (line 456-460):
   ```python
   leader_positions = getattr(self, '_leader_positions', None)
   if leader_positions:
       joint_action = {f"{name}.pos": pos for name, pos in leader_positions.items()}
       self.robot.send_action(joint_action)  # <-- No locked joint enforcement!
   ```

## Solution

Modify `RobotEnv.step()` to enforce locked joints when sending leader positions to follower:

```python
leader_positions = getattr(self, '_leader_positions', None)
if leader_positions:
    joint_action = {f"{name}.pos": pos for name, pos in leader_positions.items()}

    # Enforce locked joints if configured
    locked_joints = getattr(self.robot.config, 'locked_joints', None)
    if locked_joints:
        motor_names = list(self.robot.bus.motors.keys())
        for joint_idx in locked_joints:
            if joint_idx < len(motor_names):
                motor_name = motor_names[joint_idx]
                joint_action[f"{motor_name}.pos"] = 90.0  # Lock at 90°

    self.robot.send_action(joint_action)
```

## Impact

- 18 episodes (793 frames) recorded with incorrect joint configuration
- Must re-record with locked joints enforced
- All previous HIL-SERL training with this dataset is invalid

## Files to Modify

1. `lerobot/src/lerobot/scripts/rl/gym_manipulator.py`
   - `RobotEnv.step()` - enforce locked joints when mirroring leader positions

## Verification

After fix, verify that recorded data has:
- Joint 3 mean ≈ 90° with std < 1°
- Joint 4 mean ≈ 90° with std < 1°
