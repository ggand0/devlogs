# 062: Privileged State Bug in Image-Based RL

## Summary

Image-based DrQ-v2 training accidentally included `cube_pos` in `low_dim_state`, defeating the purpose of learning from pixels for sim-to-real transfer.

## Bug Discovery

Checkpoint analysis revealed:
```
low_dim_obs.0.weight: torch.Size([50, 63])  # 63 = 21 dims × 3 frames
```

Expected for true image-based RL: 54 dims (18 × 3 frames) or 0 dims (pure image).

## Root Cause

`WristCameraWrapper` in `src/training/so101_factory.py` passed full observation:

```python
# BUG: passes all 21 dims including cube_pos
return {
    "rgb": img,
    "low_dim_state": obs.astype(np.float32),
}
```

Observation layout (21 dims):
| Index | Name | Dims | Available on Real Hardware |
|-------|------|------|---------------------------|
| 0:6 | joint_pos | 6 | Yes (encoders) |
| 6:12 | joint_vel | 6 | Yes (encoders) |
| 12:15 | gripper_xyz | 3 | Yes (FK) |
| 15:18 | gripper_euler | 3 | Yes (FK) |
| 18:21 | cube_pos | 3 | **No (sim only)** |

## Impact

- Agent learned to rely on ground-truth cube position
- 100% success rate was misleading - policy won't transfer to real robot
- All checkpoints from image-based training are unusable for sim-to-real

## Fix

Filter out `cube_pos`, keep only proprioception (18 dims):

```python
PROPRIOCEPTION_DIM = 18

def observation(self, obs):
    ...
    proprioception = obs[:self.PROPRIOCEPTION_DIM].astype(np.float32)
    return {"rgb": img, "low_dim_state": proprioception}
```

## Commit

- Bug introduced: `4cb0ba9` (Dec 31, 2025) - "Add RoboBase integration for image-based RL training"
- Bug fixed: `e3e73f3` (Jan 6, 2026) - "Fix privileged state leak in image-based RL"

## Verification

```python
# After fix
obs['low_dim_state'].shape  # (3, 18) = 54 total dims
# Before fix (buggy)
obs['low_dim_state'].shape  # (3, 21) = 63 total dims
```

## Next Steps

1. Retrain from scratch with fixed wrapper (~2M steps)
2. Verify checkpoint has 54-dim low_dim_obs, not 63-dim
3. Evaluate sim-to-real transfer capability
