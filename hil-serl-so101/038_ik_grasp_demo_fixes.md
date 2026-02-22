# Devlog 038: IK Grasp Demo Fixes for Corrected Joint Assembly

**Date**: 2025-01-14
**Status**: Complete

## Overview

Fixed `scripts/ik_grasp_demo.py` after physical correction of joint 4 (wrist_roll) assembly. The joint was previously mounted upside-down and has now been fixed to match the MuJoCo model orientation.

## Background

Joint 4 (wrist_roll) was assembled 180 degrees flipped from the correct orientation. After physical correction:
- Static finger now comes on the bottom side in default wrist rotation (matches MuJoCo pose)
- Wrist roll value needed to change from `-π/2` to `+π/2` to match MuJoCo convention

## Changes Made

### 1. Wrist Roll Correction

Updated all wrist_roll values from `-π/2` to `+π/2`:

```python
# Before (compensating for upside-down assembly)
current_joints[4] = -np.pi / 2

# After (matches MuJoCo convention)
current_joints[4] = np.pi / 2
```

Applied in:
- `move_to_position()` function (current_joints and target_joints)
- Initial top-down wrist orientation setup

### 2. Increased Grasp Heights

The IK motion path can swing low during approach. Increased heights to prevent ground collision:

| Parameter | Before | After | Total Height |
|-----------|--------|-------|--------------|
| HEIGHT_OFFSET | 30mm | 60mm | 80mm (above position) |
| GRASP_HEIGHT_SAFETY | 15mm | 50mm | 70mm (grasp position) |

### 3. Gripper Closure

Changed `GRIPPER_CLOSED` from `-0.8` to `-1.0` for full closure:

```python
# Gripper mapping: policy (-1 to 1) → LeRobot (0 to 100)
# -0.8 → 10% open (90% closed)
# -1.0 → 0% open (fully closed)
GRIPPER_CLOSED = -1.0
```

### 4. Fixed Arm Drift During Gripper Operations

Previously, the close/release steps read `current_joints` each iteration, causing arm drift:

```python
# Before (arm drifts)
for grip in np.linspace(...):
    current_joints = robot.get_joint_positions_radians()
    robot.send_action(current_joints, grip)

# After (arm stays fixed)
grasp_joints = robot.get_joint_positions_radians()
for grip in np.linspace(...):
    robot.send_action(grasp_joints, grip)
```

Applied to both:
- Step 3: Close gripper (save `grasp_joints` before closing)
- Step 5: Release gripper (save `lift_joints` before opening)

## New Script: capture_gripper_states.py

Added utility script to capture wrist camera images at different gripper positions for calibration/debugging:

```bash
uv run python scripts/capture_gripper_states.py --output_dir ./gripper_images
```

Captures images at:
- `closed`: -1.0 (0% physical)
- `half_open`: 0.0 (50% physical)
- `fully_open`: 1.0 (100% physical)

## Usage

```bash
# Run grasp demo with default settings
uv run python scripts/ik_grasp_demo.py --output recordings/grasp_demo.mp4

# With custom gripper open position
uv run python scripts/ik_grasp_demo.py --gripper_open 0.5

# With half-open gripper (50% physical)
uv run python scripts/ik_grasp_demo.py --half_open
```

## Files Modified

- `scripts/ik_grasp_demo.py` - Wrist roll fix, height adjustments, gripper closure, arm drift fix
- `scripts/capture_gripper_states.py` - New gripper calibration utility

## Related

- Physical hardware fix: Joint 4 (wrist_roll) reassembled to correct orientation
- Devlog 037: Genesis DrQ-v2 inference (wrist_roll compensation was `-π/2` for flipped joint)
