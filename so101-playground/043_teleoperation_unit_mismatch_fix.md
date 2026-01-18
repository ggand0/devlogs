# Devlog 043: Teleoperation Unit Mismatch Fix

**Date**: 2025-01-18
**Status**: Complete

## Problem

During HIL-SERL demonstration recording, the follower arm would not mirror the leader arm movements. The IK reset completed successfully and leader torque was disabled, but the user could not control the robot during teleoperation.

Episodes were recorded but extremely short (small video files), ending prematurely.

## Root Cause Analysis

The leader and follower arms were using different unit systems for joint positions:

| Component | `use_degrees` | MotorNormMode | Unit Range |
|-----------|---------------|---------------|------------|
| Leader (teleop) | `true` | `DEGREES` | -180° to 180° |
| Follower (robot) | `false` | `RANGE_M100_100` | -100 to 100 |

### The Bug Flow

1. Leader's `sync_read("Present_Position")` returns **degree values** (e.g., 90.0°)
2. These values are stored in `_leader_positions` for joint mirroring
3. Follower's `sync_write("Goal_Position")` receives 90.0
4. Follower's `_unnormalize()` interprets 90.0 as a **normalized range value** (90% of range)
5. Result: **Completely wrong joint positions** sent to follower motors

### Code Path

**Intervention handler** (`gym_manipulator.py:1795-1815`):
```python
# Leader reads position in degrees (use_degrees: true)
leader_pos_dict = self.robot_leader.bus.sync_read("Present_Position")
# Stores degree values directly
self.unwrapped._leader_positions = {name: leader_pos_dict[name] for name in leader_pos_dict}
```

**RobotEnv step** (`gym_manipulator.py:456-461`):
```python
leader_positions = getattr(self, '_leader_positions', None)
if leader_positions:
    # Sends degree values to follower configured with RANGE_M100_100
    joint_action = {f"{name}.pos": pos for name, pos in leader_positions.items()}
    self.robot.send_action(joint_action)  # Wrong interpretation!
```

**Motor bus normalization** (`motors_bus.py:776-803`):
```python
def _normalize(self, ids_values):
    if self.motors[motor].norm_mode is MotorNormMode.DEGREES:
        # Returns degrees: (val - mid) * 360 / max_res
        normalized_values[id_] = (val - mid) * 360 / max_res
    elif self.motors[motor].norm_mode is MotorNormMode.RANGE_M100_100:
        # Returns -100 to 100 range
        norm = (((bounded_val - min_) / (max_ - min_)) * 200) - 100
```

## Fix Applied

Changed `record_config.json` to use matching units:

```json
{
    "robot": {
        "type": "so101_follower_end_effector",
        "use_degrees": true,  // Changed from false
        ...
    },
    "teleop": {
        "type": "so101_leader",
        "use_degrees": true   // Already true
    }
}
```

Now both leader and follower use `MotorNormMode.DEGREES`, so:
- Leader reads 90.0°
- Follower receives 90.0°
- Follower's `_unnormalize()` converts 90.0° back to raw motor position correctly

## Why This Happened

The `so101_follower_end_effector` robot type was designed for IK control mode where:
- Actions are delta XYZ end-effector movements
- Joint positions are read/written in radians internally by MuJoCo
- `use_degrees: false` with `RANGE_M100_100` normalizes joint positions for the policy

For **teleoperation/recording**, the wrapper falls back to direct joint mirroring because no kinematics path exists for leader→follower. This requires matching unit systems.

## Files Modified

- `outputs/hilserl_drqv2/record_config.json` - Set `robot.use_degrees: true`

## Key Insight

When using `so101_follower_end_effector` for teleoperation recording (not IK inference), the `use_degrees` setting must match the leader's setting. The IK path handles its own unit conversions internally, but the fallback joint mirroring path requires consistent units.

## Related

- Devlog 042: DrQ-v2 Encoder Synchronization Fix
- The train_config.json can keep `use_degrees: false` since training uses IK actions, not joint mirroring
