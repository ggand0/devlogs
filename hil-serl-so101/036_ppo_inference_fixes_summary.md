# Devlog 036: PPO Inference Script Fixes - Executive Summary

**Date**: 2025-01-12
**Status**: Complete

## Overview

Fixed critical issues in `ppo_inference.py` that prevented Genesis PPO policies from executing correct motions on the real SO-101 robot. The robot now successfully performs forward approach motion.

## Issues Fixed

### 1. Coordinate Frame Mismatch (Genesis vs MuJoCo)

**Problem**: Genesis and MuJoCo use different coordinate systems.

| Direction | Genesis Frame | MuJoCo Frame |
|-----------|--------------|--------------|
| Forward   | **-Y**       | **+X**       |
| Sideways  | X            | Y            |
| Up/Down   | Z            | Z            |

**Symptom**: Robot drifted sideways instead of moving forward.

**Fix**: Added `--genesis_to_mujoco` flag that transforms actions:
```python
if args.genesis_to_mujoco:
    genesis_x, genesis_y, genesis_z = delta_xyz
    delta_xyz = np.array([
        -genesis_y,  # MuJoCo X (forward) = -Genesis Y
        genesis_x,   # MuJoCo Y (sideways) = Genesis X
        genesis_z,   # Z unchanged
    ])
```

### 2. IK Joint Offset Mismatch

**Problem**: IK was computing from wrong starting position because sensor offset wasn't applied.

**Symptom**: Robot moved backward when commanded forward; Z motion worked but X motion didn't.

**Root Cause**: The elbow_flex sensor reads ~12.5° more bent than actual physical position. IK used raw sensor values, causing:
- IK thought EE was at position A
- Actual EE was at position B
- IK computed motion from A → wrong direction

**Fix**: Apply offset correction before IK, convert back for robot commands:
```python
current_joints_raw = robot.get_joint_positions_radians()
current_joints = apply_joint_offset(current_joints_raw)  # Correct for sensor offset

target_joints = ik.cartesian_to_joints(delta_xyz, current_joints, ...)

# Convert back to raw joints for robot command
target_joints[2] -= ELBOW_FLEX_OFFSET_RAD
```

### 3. Wrist Joint Coupling

**Problem**: Only locking wrist_roll (joint 4) during IK caused wrist_pitch (joint 3) changes that coupled into backward X motion.

**Fix**: Lock both wrist joints during inference:
```python
locked_joints=[3, 4]  # Was [4] only
```

### 4. Action Scale Too Small

**Problem**: With `action_scale=0.01`, joint changes were ~0.004 rad (0.2°), potentially below servo resolution.

**Fix**: Use `action_scale=0.05` for visible motion.

## Test Results

**Before fixes** (mock camera, no transform):
```
Step 0:  ee=[0.272,-0.014,0.030]
Step 10: ee=[0.272,-0.014,0.013]  # Only Z changed, X stuck
```

**After fixes** (real camera, with transform):
```
Step 0:  ee=[0.272,-0.014,0.032]
Step 10: ee=[0.271,-0.010,0.006]
Step 20: ee=[0.284,-0.010,0.004]
Step 30: ee=[0.307,-0.008,0.006]  # +35mm forward
```

## Working Command

```bash
uv run python scripts/ppo_inference.py \
    --checkpoint /path/to/checkpoint.pt \
    --genesis_to_mujoco \
    --camera_index 1 \
    --action_scale 0.05 \
    --episode_length 40
```

## Files Modified

- `scripts/ppo_inference.py`:
  - Added `--genesis_to_mujoco` for coordinate frame transformation
  - Fixed IK to use `apply_joint_offset()` before computing targets
  - Convert target joints back to raw space for robot commands
  - Lock both wrist joints [3, 4] instead of just [4]
  - Added IK debug output (current_ee, target_ee, joint_diff)
  - Show full EE position in status output

## Remaining Issues

1. **Wrist Configuration Mismatch**: Genesis trains with `joint[4]=+π/2`, real robot uses `-π/2`. This affects euler angles in low_dim_state. Current checkpoint may not transfer well - need checkpoint trained with correct wrist config.

2. **Early Checkpoint Behavior**: The 100k checkpoint only learned "reach forward and down" behavior. Full grasp-and-lift requires more training.

3. **Visual Input**: Policy produces similar actions with real camera vs random noise, suggesting it relies mainly on proprioception at this training stage.

## Key Learnings

1. **Always match coordinate frames** between training and deployment environments
2. **Sensor offsets matter for IK** - if FK uses corrected joints, IK must too
3. **Lock all relevant joints** when using partial IK to prevent coupling
4. **Action scale affects servo execution** - too small may not execute
