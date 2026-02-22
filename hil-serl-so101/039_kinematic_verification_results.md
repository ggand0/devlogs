# Devlog 039: Kinematic Verification Results

**Date**: 2025-01-16
**Status**: Complete

## Overview

Verified MuJoCo forward kinematics against physical robot measurements after correcting the 180-degree flipped wrist assembly (joint 4-5 connection).

## Test Setup

- Used `scripts/verify_kinematics.py --current-pose` to read current joint positions and compute FK
- Reference point: `front_base_marker` at [8.0, 0.0, 1.5] cm (front edge of robot base, on table)
- Model: `lift_cube_calibration.xml`

## Results

### Joint Readings
```
Current joints (deg): [1.54, 32.13, -18.46, 74.55, -0.57]
```

### Computed vs Measured EE Position (from front_base_marker)

| Axis | Computed | Measured | Error |
|------|----------|----------|-------|
| X (forward) | +21.00 cm | +22.3 cm | **1.3 cm** |
| Y (left) | -0.69 cm | ~0 cm | ~0.7 cm |
| Z (up) | -0.39 cm | ~0 cm | ~0.4 cm |

### Gripperframe Offset

The `gripperframe` site is 9.8 cm from the gripper body, while the actual fingertips are at 10.4 cm. This accounts for ~0.6 cm offset between computed TCP and physical fingertip position.

## Error Analysis

- **X-axis error (1.3 cm)**: Could be from joint calibration offsets or small link length discrepancies in URDF
- **Y-axis error (~0.7 cm)**: Within acceptable range, likely joint reading noise
- **Z-axis error (~0.4 cm)**: Negligible, within measurement tolerance

## Conclusion

Kinematic accuracy is within **~1.5 cm** which is acceptable for the cube picking task (3x3x3 cm cube). The 180-degree wrist assembly fix is verified - FK directions are now correct.

## Script Updates

Added `--current-pose` flag to `verify_kinematics.py`:
- Reads current joint positions without moving the robot
- Computes FK and shows distances from `front_base_marker` reference point
- Safe operation - no motor commands issued
