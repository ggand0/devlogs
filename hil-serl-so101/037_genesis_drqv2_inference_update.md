# Devlog 037: Genesis DrQ-v2 Inference Update

**Date**: 2025-01-12
**Status**: Complete

## Overview

Updated `scripts/rl_inference.py` to support Genesis-trained DrQ-v2 policies by porting critical fixes from `ppo_inference.py`. The script now handles the coordinate system differences and sensor calibration issues required for Genesis-to-real robot transfer.

## Background

Genesis and MuJoCo use different coordinate systems:
- **Genesis**: X=sideways, -Y=forward, Z=up (robot at y=0.314 facing -Y)
- **MuJoCo**: X=forward, Y=sideways, Z=up (robot at origin facing +X)

Additionally, the elbow_flex sensor has a ~12.5° offset that affects IK accuracy.

## Changes Made

### New CLI Arguments

```python
# Genesis-specific flags
--genesis_to_mujoco    # Enable coordinate transform only
--genesis_mode         # Enable all Genesis fixes (recommended)
--pick101_root         # Configurable pick-101 path (was hardcoded)

# Debug options (ported from ppo_inference.py)
--debug_state          # Verbose state/action/IK debug output
--save_obs             # Save observation images
--save_obs_video       # Save observations as video
--rotate_image         # Rotate camera images (0/90/180/270)

# Sim2real diagnostic
--sim_frames_dir       # Pre-recorded sim frames for testing
--sim_states_file      # Pre-recorded sim states for testing
```

### Critical Genesis Fixes

#### 1. Coordinate Transform (lines 459-468)

```python
if use_genesis:
    genesis_x, genesis_y, genesis_z = delta_xyz
    delta_xyz = np.array([
        -genesis_y,  # MuJoCo X (forward) = -Genesis Y
        genesis_x,   # MuJoCo Y (sideways) = Genesis X
        genesis_z,   # Z unchanged
    ])
```

#### 2. Joint Offset Correction (lines 253-265)

```python
ELBOW_FLEX_OFFSET_RAD = np.deg2rad(-12.5)

def apply_joint_offset(joints: np.ndarray) -> np.ndarray:
    corrected = joints.copy()
    corrected[2] += ELBOW_FLEX_OFFSET_RAD
    return corrected
```

Applied before IK computation, undone before sending to robot.

#### 3. Wrist Joint Locking (line 483)

```python
locked_joints=[3, 4] if use_genesis else [4]
```

Genesis mode locks both wrist joints to prevent coupling issues.

#### 4. Reset Height (lines 239-247)

```python
if use_genesis:
    HEIGHT_OFFSET = 0.0   # Genesis: at grasp height (0.02m)
    RESET_GRIPPER = 0.3   # Genesis uses partially open gripper
else:
    HEIGHT_OFFSET = 0.03  # MuJoCo: 3cm above grasp height
    RESET_GRIPPER = 1.0   # Fully open
```

### Configurable pick-101 Path

Removed hardcoded path, now uses `--pick101_root` argument:

```python
pick101_root = Path(args.pick101_root)
sys.path.insert(0, str(pick101_root))
sys.path.insert(0, str(pick101_root / "external" / "robobase"))
```

### Sim2Real Diagnostic Support

Added ability to replay pre-recorded sim observations:

```python
if use_sim_obs:
    frame_idx = (global_step + step) % len(sim_frames)
    rgb_obs = np.stack([
        sim_frames[max(0, frame_idx - i)]
        for i in range(policy.frame_stack - 1, -1, -1)
    ], axis=0)
```

## Usage Examples

```bash
# Backward compatible (MuJoCo DrQ-v2)
uv run python scripts/rl_inference.py --checkpoint /path/to/mujoco_drqv2.pt

# Genesis DrQ-v2 with all fixes (recommended)
uv run python scripts/rl_inference.py --checkpoint /path/to/genesis_drqv2.pt --genesis_mode

# Genesis with debug output
uv run python scripts/rl_inference.py --checkpoint /path/to/genesis_drqv2.pt \
    --genesis_mode --debug_state --action_scale 0.05

# Sim2real diagnostic
uv run python scripts/rl_inference.py --checkpoint /path/to/genesis_drqv2.pt \
    --genesis_mode --sim_frames_dir ./sim_episode/ --sim_states_file ./sim_states.npy
```

## Key Differences: Genesis vs MuJoCo Mode

| Feature | MuJoCo (default) | Genesis (`--genesis_mode`) |
|---------|------------------|---------------------------|
| Coordinate transform | None | Genesis→MuJoCo |
| Joint offset | None | elbow_flex -12.5° |
| Locked joints | `[4]` | `[3, 4]` |
| Reset height | 0.05m (above cube) | 0.02m (grasp height) |
| Reset gripper | 1.0 (fully open) | 0.3 (partially open) |

## Files Modified

- `scripts/rl_inference.py` - Added Genesis support while maintaining backward compatibility

## Related Devlogs

- Devlog 036: PPO inference fixes (coordinate transform, joint offset discovery)
- Devlog 032: Kinematic verification (elbow_flex offset measurement)
