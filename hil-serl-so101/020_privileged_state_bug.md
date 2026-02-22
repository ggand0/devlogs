# Devlog 020: Privileged State Bug in Image RL

## Summary

Real robot inference failed because the trained policy secretly relied on cube position - privileged information unavailable on the real robot.

## The Bug

Training checkpoint `runs/image_rl/.../best_snapshot.pt` expected 63-dim `low_dim_state`:

```
low_dim_obs.0.weight: torch.Size([50, 63])
```

Breakdown: `[joint_pos(6) + joint_vel(6) + gripper_xyz(3) + gripper_euler(3) + cube_pos(3)] × 3 frames = 63`

**cube_pos is sim-only privileged info.**

## Root Cause

In pick-101's `src/envs/wrappers/image_obs.py`:

```python
def observation(self, obs):
    self._renderer.update_scene(self.unwrapped.data, camera=self.camera)
    img = self._renderer.render()
    return {"rgb": img, "low_dim_state": obs}  # ← BUG: passes full state
```

The wrapper was meant to make obs image-only, but leaked the original state including cube_pos.

## How Inference Handled It

In `src/deploy/policy.py`, `LowDimStateBuilder` was set with `include_cube_pos=False`:

```python
state_builder = LowDimStateBuilder(include_cube_pos=False)
```

This zero-padded the cube_pos dimensions:

```python
if self.include_cube_pos:
    parts.append(cube_pos)
else:
    parts.append(np.zeros(3, dtype=np.float32))  # ← zeros instead of real pos
```

## Result

Policy learned to rely on cube_pos for precise grasping. At inference:
- We fed zeros for cube_pos
- Policy moved vaguely toward cube area (from image)
- But couldn't grasp accurately (missing precise position info)

## Fix

1. Fix `image_obs.py` in pick-101:
   ```python
   return {"rgb": img}  # Image only, no low_dim_state
   ```

2. Retrain DrQ-v2 from scratch (~2M steps)

3. Re-run inference with new checkpoint

## Lesson

Always verify observation space matches between training and deployment. Check checkpoint weight shapes to detect dimension mismatches.
