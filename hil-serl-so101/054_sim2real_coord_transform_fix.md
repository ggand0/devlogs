# Devlog 054: Sim2Real Coordinate Transform Fix

## Problem

DrQ-v2 policy trained in pick-101 MuJoCo sim (70% success in sim) was not working on real robot. The arm "drifted forward while the gripper opened/closed randomly" instead of grasping the cube.

## Investigation

After verifying segmentation class remapping was correct (devlog 053), investigated the coordinate system and mode flags:

1. **Genesis mode was conflating multiple behaviors**:
   - Coordinate transform (Genesis -> MuJoCo)
   - Gripper reset position (0.3 partially open)
   - Wrist locking ([3, 4] joints locked at pi/2)

2. **The seg+depth policy was trained in MuJoCo** (pick-101), not Genesis:
   - Config: `drqv2_lift_seg_depth_v19.yaml`
   - Factory: `so101_lift` (MuJoCo-based)
   - No coordinate transform needed

3. **Using `--genesis_mode` applied WRONG coordinate transform**:
   ```python
   # Genesis transform (WRONG for MuJoCo-trained policy):
   delta_xyz = [-genesis_y, genesis_x, genesis_z]
   ```
   This swapped X/Y and negated Y, causing the "drift forward" behavior.

## Root Cause

The `--genesis_mode` flag was designed for Genesis-trained policies but was being used with MuJoCo-trained policies, causing:
1. Actions in MuJoCo frame being incorrectly transformed
2. XY motion swapped/negated while Z remained correct

## Solution

Added `--mujoco_mode` flag for pick-101 trained policies:
- **No coordinate transform** (MuJoCo native coordinates)
- **Gripper = 0.3** (matches training curriculum_stage=3)
- **Wrist locked at pi/2** (top-down grasp orientation)

Separated coordinate transform from other mode settings:
```python
apply_coord_transform = use_genesis and not use_mujoco
lock_wrist = use_genesis or use_mujoco
```

## Changes

### `scripts/rl_inference_seg_depth.py`
- Added `--mujoco_mode` flag
- Separated `apply_coord_transform` from `lock_wrist` logic
- Updated docstring with mode selection guide
- Both MuJoCo and Genesis modes now use:
  - HEIGHT_OFFSET = 0.03
  - RESET_GRIPPER = 0.3
  - locked_joints = [3, 4]

## Usage

```bash
# For pick-101 MuJoCo-trained policies (seg+depth, RGB, etc.)
uv run python scripts/rl_inference_seg_depth.py \
    --checkpoint ~/ggando/ml/pick-101/runs/seg_depth_rl/20260119_192807/snapshots/1200000_snapshot.pt \
    --seg_checkpoint ~/ggando/ml/pick-101/outputs/efficientvit_seg_merged/best-v1.ckpt \
    --mujoco_mode \
    --cube_x 0.25 --cube_y 0.0

# For Genesis-trained policies (requires coordinate transform)
uv run python scripts/rl_inference_seg_depth.py \
    --checkpoint /path/to/genesis_policy.pt \
    --seg_checkpoint /path/to/seg_checkpoint.ckpt \
    --genesis_mode
```

## Lessons Learned

1. When sim2real transfer fails with "random/drifting" behavior, check coordinate systems
2. Mode flags should be orthogonal - don't conflate coordinate transform with other settings
3. Document which simulator each policy was trained in to select correct mode
