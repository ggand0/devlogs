# Devlog 055: Observation Format Fix (Stack vs Concatenate)

## Problem

After fixing the coordinate transform issue (devlog 054), the seg+depth policy still wasn't working. The robot output Z=-1.0 (always down) and gripper=0.9+ (always open) at every step.

## Investigation

Initial attempt used `np.concatenate` to pre-flatten observations to `(6, 84, 84)`. This caused a cryptic error:
```
AssertionError('Expected shape (V, C, H, W), but got (1, np.int64(504), 84)')
```

After adding view dimension `(1, 6, 84, 84)`, the policy ran but produced nonsensical actions.

**Root cause discovered** by reading robobase source code:

The agent's `flatten_time_dim_into_channel_dim` function expects:
- Input: `(batch, views, frame_stack, channels, H, W)` = `(1, 1, 3, 2, 84, 84)`
- Output: `(batch, views, frame_stack*channels, H, W)` = `(1, 1, 6, 84, 84)`

We were pre-flattening to `(1, 1, 6, 84, 84)`, then the agent tried to flatten again:
- Interpreted as `bs=1, v=1, t=6, ch=84`
- Result: `tensor.view(bs, v, t*ch, *tensor.shape[4:])` = `(1, 1, 504, 84)` - WRONG!

## Root Cause

The `frame_stack_on_channel=True` flag means the **agent** handles flattening internally. We should NOT pre-flatten.

1. **observation_space shape**: Should be `(frame_stack, channels, H, W)` = `(3, 2, 84, 84)`
2. **Frame stacking**: Use `np.stack()` to get `(3, 2, 84, 84)`, NOT `np.concatenate()` to get `(6, 84, 84)`
3. **Tensor shape**: `(batch, frame_stack, channels, H, W)` = `(1, 3, 2, 84, 84)` - agent adds view dim internally

## Solution

### `src/deploy/policy.py`
```python
# observation_space: let agent flatten internally
observation_space = {
    "rgb": {"shape": (frame_stack, obs_channels, image_size, image_size)},  # (3, 2, 84, 84)
}

# get_action: add ONLY batch dimension (agent adds view dimension via stack_tensor_dictionary)
obs_tensor = torch.from_numpy(seg_depth_norm).unsqueeze(0).to(self.device)
# Shape: (1, 3, 2, 84, 84) - agent adds view -> (1, 1, 3, 2, 84, 84) -> flattens to (1, 1, 6, 84, 84)
```

### `scripts/rl_inference_seg_depth.py`
```python
# Use np.stack (NOT np.concatenate!)
seg_depth_obs = np.stack(list(frame_buffer), axis=0)  # (3, 2, 84, 84)
```

## Changes

- `src/deploy/policy.py`: observation_space = `(3, 2, 84, 84)`, tensor = `(1, 3, 2, 84, 84)` (batch only)
- `scripts/rl_inference_seg_depth.py`: Reverted to `np.stack`
- `src/deploy/perception.py`: Reverted both preprocessors to `np.stack`

## Key Insight

When `frame_stack_on_channel=True`:
- **Training**: FrameStack wrapper outputs `(3, 2, 84, 84)`, agent flattens to `(6, 84, 84)` internally
- **Inference**: Must provide `(3, 2, 84, 84)`, let agent flatten

The view axis is added by `stack_tensor_dictionary` in `_act_extract_rgb_obs`:
```python
def _act_extract_rgb_obs(self, observations):
    rgb_obs = extract_many_from_spec(observations, r"rgb.*")
    rgb_obs = stack_tensor_dictionary(rgb_obs, 1)  # adds view axis
    if self.frame_stack_on_channel:
        rgb_obs = flatten_time_dim_into_channel_dim(rgb_obs, has_view_axis=True)
```

So we provide `(batch, T, C, H, W)` = `(1, 3, 2, 84, 84)`, NOT `(batch, views, T, C, H, W)`.

## Additional Fix: Double Normalization

After fixing the shape, the robot still output Z=-1.0 and gripper=0.9 constantly. Investigation revealed **double normalization**:

```python
# SegDepthPolicyRunner (WRONG - was normalizing)
seg_depth_norm = seg_depth.astype(np.float32) / 255.0  # -> [0, 1]
# Then encoder normalizes again: [0, 1] / 255 -> [0, 0.004]

# PolicyRunner for RGB (CORRECT - no normalization)
rgb_tensor = torch.from_numpy(rgb).unsqueeze(0).float().to(self.device)  # [0, 255]
# Encoder normalizes: [0, 255] / 255 -> [0, 1]
```

**Fix**: Remove `/255.0` from `SegDepthPolicyRunner.get_action()`:
```python
obs_tensor = torch.from_numpy(seg_depth.astype(np.float32)).unsqueeze(0).to(self.device)
```

## Verification

```bash
uv run python scripts/rl_inference_seg_depth.py \
    --checkpoint ~/ggando/ml/pick-101/runs/seg_depth_rl/.../snapshot.pt \
    --seg_checkpoint ~/ggando/ml/pick-101/outputs/efficientvit_seg_merged/best-v1.ckpt \
    --mujoco_mode --debug_state
```

Expected:
```
obs shape: (3, 2, 84, 84) (expected: (3, 2, 84, 84))
```
