# IK Controller Implementation for SO-101 Real Robot

## Overview

The IK controller uses MuJoCo purely as a kinematics solver (not simulator) for real robot deployment. It computes inverse kinematics via damped least-squares using MuJoCo's Jacobian computation.

## Architecture

```
Policy Action (delta XYZ)
    → IK Controller (Jacobian-based)
    → Target Joint Positions (radians)
    → LeRobot Normalized (-100 to 100)
    → Real Robot
```

## Coordinate System

- **Z-up**: MuJoCo uses Z-up coordinate system
- **Origin**: Robot base
- **Home position** (all joints = 0): EE at approximately `[0.39, 0, 0.23]` meters

## Workspace Reference

| Z Height | Description |
|----------|-------------|
| 0.23m | Home position (arm extended) |
| 0.05m | Slightly above tabletop |
| 0.01m | Grab height for 2cm cube (cube center at ~1cm from table) |
| 0.0m | Tabletop level |

Example: `[0.0, 0.0, 0.01]` successfully grabs a 2cm cube on the table.

## Joint Ranges (from Calibration)

Derived from LeRobot calibration JSON using Feetech servo spec (4096 encoder units = 2π radians):

| Joint | Encoder Range | Radians | Degrees |
|-------|--------------|---------|---------|
| shoulder_pan | 1030-3460 | ±1.86 | ±107° |
| shoulder_lift | 827-3225 | ±1.84 | ±105° |
| elbow_flex | 925-3161 | ±1.71 | ±98° |
| wrist_flex | 954-3305 | ±1.80 | ±103° |
| wrist_roll | 33-4009 | ±3.05 | ±175° |

## Position Conversion

LeRobot uses normalized positions (-100 to 100). Conversion to radians:

```python
# Normalized → Encoder → Radians
encoder = (normalized + 100) / 200 * (range_max - range_min) + range_min
mid_encoder = (range_min + range_max) / 2
radians = (encoder - mid_encoder) / (4096 / 2π)
```

Key insight: Mid-point of encoder range = 0 radians.

## Gripper Convention

| Format | Closed | Open |
|--------|--------|------|
| Policy | -1 | +1 |
| LeRobot | 0 | 100 |

Conversion: `lerobot = (policy + 1) / 2 * 100`

## IK Algorithm

Damped least-squares IK:
1. Compute position error: `e = target_pos - current_pos`
2. Compute Jacobian via `mj_jacSite()`
3. Solve: `dq = (J^T J + λ²I)^{-1} J^T e`
4. Clamp: `dq = clip(dq, -max_dq, max_dq)`
5. Apply: `q_target = q_current + dq`
6. Enforce joint limits

Parameters:
- `damping = 0.1` (singularity robustness)
- `max_dq = 0.5` rad/step (velocity limit)

## Usage Modes

### Delta-based (Recommended)
Works without perfect FK calibration. Uses local Jacobian.
```python
target_joints = ik.cartesian_to_joints(
    delta_xyz,      # Normalized [-1, 1]
    current_joints,
    action_scale=0.02  # 2cm per unit
)
```

### Absolute positioning
Requires calibrated FK matching reality.
```python
target_joints = ik.compute_ik(target_pos, current_joints)
```

## Files

- `src/deploy/controllers/ik_controller.py` - IK controller implementation
- `src/deploy/robot.py` - LeRobot interface with position conversion
- `scripts/test_ik_motion.py` - Interactive IK test (WASD controls)

## Calibration File

Location: `/home/gota/.cache/huggingface/lerobot/calibration/robots/so101_follower/ggando_so101_follower.json`

Contains per-joint:
- `range_min`, `range_max`: Encoder limits
- `homing_offset`: Used by LeRobot for home position
- `drive_mode`: Motor configuration
