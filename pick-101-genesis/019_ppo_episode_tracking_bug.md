# Devlog 019: PPO Episode Tracking Bug Fix

**Date**: 2026-01-11
**Status**: FIXED - Needs verification

## Problem

Training metrics showed suspicious discrepancy:

| Metric | Training Rollout | Evaluation |
|--------|-----------------|------------|
| ep_rew_mean | 8.6 | 852.2 |
| ep_len_mean | 52.5 | 300.0 |
| ep_success_rate | 0% | 100% |

Video showed successful grasping, but rollout metrics were way off.

## Root Cause

Two bugs in `src/training/ppo.py`:

### Bug 1: Episode tracking reset every rollout

```python
def collect_rollout(self, num_steps: int, ...):
    # BUG: These get reset to zero EVERY rollout
    ep_rewards = torch.zeros(num_envs, device=self.device)
    ep_lengths = torch.zeros(num_envs, device=self.device, dtype=torch.int32)
```

With config:
- `num_steps = 128` (steps per env per rollout)
- `max_episode_steps = 300`
- Episodes span ~2.3 rollouts (300 / 128)

**What happened:**
1. Rollout 1: env.step_count goes 0→127, ep_lengths=128, no episode completes
2. Rollout 2: **ep_lengths reset to 0**, env.step_count continues 128→255
3. Rollout 3: **ep_lengths reset to 0**, env.step_count 256→383, done triggers at 300
4. Recorded ep_length = 300 - 256 = **44 steps** (only the fragment in rollout 3)

Same for rewards - only final fragment recorded (~8.6 instead of ~400-850).

### Bug 2: All envs reset when any env done

```python
if done_mask.any():
    self.env.reset()  # BUG: Resets ALL 16 envs, not just the done ones
```

This caused:
- Non-done envs lose their episode progress
- Corrupted episode tracking for non-done envs
- Wasted computation re-initializing envs that weren't done

## Fix Applied

### ppo.py changes

1. **Persistent episode tracking** (lines 167-171):
```python
def __init__(self, ...):
    ...
    # Persistent episode tracking (survives across rollouts)
    self.ep_rewards = torch.zeros(env.num_envs, device=device)
    self.ep_lengths = torch.zeros(env.num_envs, device=device, dtype=torch.int32)
```

2. **Use persistent state** (lines 230-258):
```python
# Track episode stats (using persistent state)
self.ep_rewards += reward_val
self.ep_lengths += 1
...
ep_infos.append({
    'reward': self.ep_rewards[env_idx].item(),
    'length': self.ep_lengths[env_idx].item(),
    ...
})
self.ep_rewards[env_idx] = 0
self.ep_lengths[env_idx] = 0
```

3. **Only reset done envs** (lines 266-278):
```python
if done_mask.any():
    done_env_ids = torch.where(done_mask)[0]
    self.env.reset(env_ids=done_env_ids)  # Only reset done envs
```

### lift_cube_env.py changes

1. **Partial cube reset** (lines 458-467):
```python
# Reset cube position only for specified envs
cube_spawn = torch.tensor(self.cube_spawn_pos, device=self.device)
current_cube_pos = self.cube.get_pos()
current_cube_quat = self.cube.get_quat()
current_cube_pos[env_ids] = cube_spawn
current_cube_quat[env_ids] = torch.tensor([1.0, 0.0, 0.0, 0.0], device=self.device)
```

2. **Conditional init steps** (lines 471-482):
```python
# Only run init steps if resetting all envs (to avoid affecting non-reset envs)
resetting_all = len(env_ids_list) == self.num_envs
if resetting_all:
    # 100 physics steps for arm settling
    for _ in range(100):
        self.robot.control_dofs_position(init_target)
        self.scene.step()
```

3. **Conditional curriculum init** (lines 493-496):
```python
# Curriculum init only when resetting all envs
# (curriculum init does scene.step() which would affect non-reset envs)
if resetting_all and self.curriculum_stage >= 3:
    self._reset_gripper_near_cube(env_ids)
```

## Trade-offs and Concerns

### Partial reset skips curriculum positioning

When only some envs are done (partial reset):
- `scene.reset()` resets those envs to initial Genesis state
- Cube position is reset for those envs
- **BUT**: No arm settling (100 steps) or curriculum pre-positioning (550 steps)

This means done envs restart from whatever state `scene.reset()` gives them, NOT from the pre-grasp position that curriculum stage 3 normally provides.

**Potential issues:**
1. Done envs may start in inconsistent states
2. Training distribution changes - some episodes start pre-grasp, others don't
3. Curriculum stage 3 effectiveness reduced for partial resets

### Why this trade-off exists

The curriculum init calls `scene.step()` which advances physics for ALL envs:
```python
def _reset_gripper_near_cube(self, env_ids):
    for _ in range(50):
        self.scene.step()  # Affects ALL 16 envs, not just env_ids!
    for _ in range(300):
        self.scene.step()
    for _ in range(200):
        self.scene.step()
```

If we ran this for partial resets, non-done envs would advance 550 physics steps mid-episode, corrupting their state.

### Alternative approaches (not implemented)

1. **Always reset all envs together**: Simpler but wasteful - envs that aren't done lose progress
2. **Per-env curriculum init**: Would require Genesis to support per-env scene.step(), which it doesn't
3. **Batch resets**: Accumulate done envs and reset them together periodically

## Verification Steps

1. **Check ep_len_mean**: Should now be ~300 (full episode), not ~50 (fragment)
2. **Check ep_rew_mean**: Should match eval reward (~400-850), not fragment (~8)
3. **Watch for instability**: Partial resets skipping curriculum might cause issues
4. **Compare learning curves**: Before vs after fix

## Expected Results After Fix

| Metric | Before Fix | After Fix |
|--------|-----------|-----------|
| ep_len_mean | ~52 | ~300 |
| ep_rew_mean | ~8 | ~400-850 |
| Rollout/Eval gap | 100x | ~1x |

## Files Modified

- `src/training/ppo.py` - Episode tracking and reset logic
- `src/envs/lift_cube_env.py` - Partial reset handling

## Open Questions

1. Does `scene.reset(envs_idx=[...])` properly reset robot arm state, or just positions?
2. Will skipping curriculum init for partial resets hurt learning significantly?
3. Should we add a "batch reset" mode that waits for multiple envs to be done?

---

## Addendum: Multi-Cube Video Rendering Bug

**Date**: 2026-01-12
**Status**: FIXED

### Problem

Training eval videos showed multiple red cubes in the scene instead of just one. This made it difficult to visually verify grasping behavior.

### Root Cause

Genesis parallel environments (`scene.build(n_envs=16)`) place ALL cubes in the same world space. The wide camera renders all of them since they're all visible in the scene.

**Why the initial fix didn't work:**

Attempted to hide cubes by moving them to `z=-10` before rendering:
```python
# This didn't work
if self.num_envs > 1:
    cube_pos_backup = self.cube.get_pos().clone()
    hidden_pos = cube_pos_backup.clone()
    hidden_pos[1:, 2] = -10.0  # Move envs 1+ below camera view
    self.cube.set_pos(hidden_pos)
    # Render here...
```

Genesis doesn't update the visual state until `scene.step()` is called, but calling `scene.step()` advances physics and corrupts the episode.

### Solution

Create a separate single-env (`num_envs=1`) environment exclusively for video recording.

**train_ppo.py changes:**

1. Create video_env at training start:
```python
# Create separate single-env for video recording (avoids multi-cube visual bug)
print("Creating video recording environment (num_envs=1)...")
video_env = LiftCubeEnv(
    model_path=cfg.env.model_path,
    num_envs=1,  # Single env for clean video
    show_viewer=False,
    use_wrist_cam=cfg.env.use_wrist_cam,
    image_size=image_size,
    device=cfg.train.device,
    max_episode_steps=cfg.env.max_episode_steps,
    action_scale=cfg.env.action_scale,
    lift_height=cfg.env.lift_height,
    hold_steps=cfg.env.hold_steps,
    reward_version=cfg.env.reward_version,
    curriculum_stage=cfg.env.curriculum_stage,
    domain_randomization=False,  # Consistent visuals for video
)
```

2. Update `record_video()` to accept `video_env`:
```python
def record_video(agent, video_env, video_path: str, device: str, max_steps: int = 150):
    """Record a split-view video of one deterministic episode."""
    video_env.reset()
    # ... inference loop using video_env ...
```

3. Call record_video with video_env:
```python
record_video(agent, video_env, str(video_path), cfg.train.device)
```

### Performance Impact

- **GPU memory**: +1-2GB for the additional Genesis environment
- **Init time**: +10s at training start
- **Training throughput**: Unaffected (video_env only used during periodic recording)

### Files Modified

- `src/training/train_ppo.py` - Added video_env creation and updated record_video calls

### Notes

- The standalone `eval_checkpoint.py` already uses `num_envs=1`, so it was unaffected
- Training-time inline eval was the issue (used 16-env training environment)
- Videos recorded with the fix will show only one cube with consistent (non-randomized) visuals
