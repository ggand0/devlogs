# Devlog 057: Real Robot Inference - Key Fixes Summary

Summary of critical fixes needed to run sim-trained DrQ-v2 policies on real SO-101 robot. Applicable to both seg+depth and RGB observations.

## 1. Coordinate Transform (devlog 054)

**Problem**: Policy trained in MuJoCo was receiving Genesis coordinate transform, causing "drift forward" behavior.

**Fix**: Use correct mode flag based on training simulator:
```bash
# MuJoCo-trained policies (pick-101)
--mujoco_mode   # No coordinate transform, native MuJoCo coordinates

# Genesis-trained policies
--genesis_mode  # Applies Genesis->MuJoCo transform: [-y, x, z]
```

**Both modes share**:
- `HEIGHT_OFFSET = 0.03` (3cm above table)
- `RESET_GRIPPER = 0.3` (partially open)
- `locked_joints = [3, 4]` (wrist joints at pi/2 for top-down grasp)

## 2. Observation Shape - No Pre-Flattening (devlog 055)

**Problem**: Pre-flattening `(3, 2, 84, 84)` to `(6, 84, 84)` caused agent to double-flatten, producing `(504, 84)`.

**Root cause**: When `frame_stack_on_channel=True`, the agent flattens internally via `flatten_time_dim_into_channel_dim()`.

**Fix**: Provide stacked frames, let agent flatten:
```python
# Frame stacking - use np.stack, NOT np.concatenate
obs = np.stack(list(frame_buffer), axis=0)  # (3, C, 84, 84)

# Tensor shape - add batch dim only, agent adds view dim
obs_tensor = torch.from_numpy(obs).unsqueeze(0).to(device)
# Shape: (1, 3, C, 84, 84) -> agent adds view -> (1, 1, 3, C, 84, 84) -> flattens to (1, 1, 3*C, 84, 84)
```

**observation_space** should match frame-stacked shape:
```python
observation_space = {
    "rgb": {"shape": (frame_stack, channels, H, W)},  # (3, 3, 84, 84) for RGB
    "low_dim_state": {"shape": (frame_stack, state_dim)},
}
```

## 3. No Double Normalization (devlog 055)

**Problem**: Pre-normalizing observations to [0,1] then encoder normalizing again made all values ~0.

**Root cause**: DrQ-v2 encoder expects uint8-range float32 and normalizes internally (`/ 255.0`).

**Fix**: Pass uint8-range values as float32, no pre-normalization:
```python
# WRONG - double normalization
obs_norm = obs.astype(np.float32) / 255.0  # -> [0, 1]
# Encoder: [0, 1] / 255 -> [0, 0.004] -- garbage!

# CORRECT - let encoder normalize
obs_tensor = torch.from_numpy(obs.astype(np.float32)).unsqueeze(0).to(device)
# Encoder: [0, 255] / 255 -> [0, 1] -- correct!
```

## 4. Segmentation Class Remapping (devlog 053)

**Problem**: Real segmentation model uses different class IDs than sim.

**Mapping** (for seg+depth policies):
| Real ID | Real Class | Sim ID | Sim Class |
|---------|------------|--------|-----------|
| 0 | unlabeled | 0 | unlabeled |
| 1 | background | 0 | (treat as unlabeled) |
| 2 | table | 1 | table |
| 3 | cube | 2 | cube |
| 4 | static_finger | 3 | robot arm |
| 5 | moving_finger | 4 | gripper |

**Fix**: Subtract 1 from non-zero classes:
```python
seg_remapped = np.where(seg_raw > 0, seg_raw - 1, 0).astype(np.uint8)
```

## 5. IK Reset Sequence

**Critical**: Proper reset to grasp-ready position before each episode.

```python
# 1. Extended forward position (clear workspace)
reset_angles = [0.0, -0.5, 0.5, pi/2, pi/2]  # Wrist locked

# 2. Set top-down wrist orientation
# Joints 3,4 at pi/2 for vertical gripper

# 3. Move to safe height above target
target_safe = [cube_x, cube_y - 0.015, 0.08]  # Y offset for wrist cam

# 4. Lower to grasp height
target_grasp = [cube_x, cube_y - 0.015, 0.05]
```

## 6. State Vector Format

**18-dim proprioception** (no cube_pos for real robot):
```python
state = [
    joint_pos[0:6],      # 6: joint positions (5 arm + 1 gripper)
    joint_vel[0:6],      # 6: joint velocities
    gripper_pos[0:3],    # 3: end-effector XYZ (from FK)
    gripper_euler[0:3],  # 3: end-effector orientation
]
# cube_pos is zero-padded or excluded
```

## Quick Checklist

Before running real inference:
- [ ] Correct mode flag (`--mujoco_mode` or `--genesis_mode`)
- [ ] Frame stacking with `np.stack`, not `np.concatenate`
- [ ] Tensor has batch dim only (agent adds view dim)
- [ ] No pre-normalization (encoder does `/255`)
- [ ] Class remapping if using segmentation
- [ ] IK reset sequence implemented
- [ ] State vector matches training format

## Files Reference

- `src/deploy/policy.py`: `PolicyRunner`, `SegDepthPolicyRunner`
- `src/deploy/perception.py`: `SegDepthPreprocessor`
- `scripts/rl_inference.py`: RGB inference script
- `scripts/rl_inference_seg_depth.py`: Seg+depth inference script
