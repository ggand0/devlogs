# Devlog 024: IK Controller Multi-Env Bug Fix

**Date:** 2026-01-13

## Critical Bug: IK Controller Only Used Env 0

### Problem

DrQ-v2 training with 16 parallel environments was completely broken. The IK controller computed joint velocities using only env 0's Jacobian and position error, then applied the **same** joint deltas to all 16 environments.

### Root Cause

In `src/controllers/ik_controller.py`:

```python
# OLD (BROKEN) CODE
def compute_joint_velocities(self, target_pos, locked_joints=None):
    jacobian = self.robot.get_jacobian(self.tcp_link)
    Jp_full = jacobian[0, :3, :]  # BUG: Only env 0's Jacobian!
    ...
    error = pos_error[0]  # BUG: Only env 0's error!
    ...
    dq = torch.zeros(5, device=self.device)  # Single dq for all envs!
```

And in `step_toward_target`:

```python
# OLD (BROKEN) CODE
for i, qpos_idx in enumerate(self.arm_qpos_indices):
    target[:, qpos_idx] = current[:, qpos_idx] + dq[i]  # Same dq applied to ALL envs!
```

### Impact

During both reset and step:
- **Env 0**: Correct IK control
- **Envs 1-15**: Completely wrong IK control - joints moved based on env 0's state

This explains:
1. Training instability after 500k steps
2. Eval episodes where agent drifts away from cube
3. Policy collapse at 800k despite strong v19 rewards

### Fix

Rewrote IK controller to compute per-env Jacobian and error:

```python
# NEW (FIXED) CODE
def compute_joint_velocities(self, target_pos, locked_joints=None):
    jacobian = self.robot.get_jacobian(self.tcp_link)  # (num_envs, 6, n_dofs)
    ...
    dq_all = torch.zeros(num_envs, 5, device=self.device)

    for env_idx in range(num_envs):
        Jp_full = jacobian[env_idx, :3, :]  # This env's Jacobian
        error = pos_error[env_idx]          # This env's error
        ...
        dq_all[env_idx, joint_idx] = dq_active[i]

    return dq_all  # (num_envs, 5)
```

And in `step_toward_target`:

```python
# NEW (FIXED) CODE
for i, qpos_idx in enumerate(self.arm_qpos_indices):
    target[:, qpos_idx] = current[:, qpos_idx] + dq[:, i]  # Per-env dq!
```

## Additional Fixes

### TensorBoard Logs

**Problem:** TB logs went to static path `./runs/genesis_rl/tb_logs` instead of run-specific directory.

**Fix:** Override `cfg.tb.log_dir` to `work_dir` before creating Logger in `genesis_workspace.py`:

```python
if hasattr(cfg, 'tb') and cfg.tb.use:
    OmegaConf.update(cfg, "tb.log_dir", str(self.work_dir))
    OmegaConf.update(cfg, "tb.name", "tb_logs")
```

### Eval Videos Split View

**Problem:** Eval videos only showed wrist camera, making it hard to understand agent behavior.

**Fix:** Updated `eval_drqv2.py` to use `get_splitview_frame()` - shows wrist cam + wide cam side by side.

## Files Modified

- `src/controllers/ik_controller.py` - Fixed multi-env IK computation
- `src/training/genesis_workspace.py` - TB logs to run directory
- `src/training/eval_drqv2.py` - Split view videos, fixed cube height tracking

## Next Steps

If training still collapses after this fix, consider:

1. **Continuous hold gradient**: Currently `hold_count` bonus only kicks in at 8cm height. Agent never experiences escalating reward during noisy exploration. Could add continuous gradient:
   ```python
   # Instead of binary threshold
   if cube_z > lift_height:
       reward += 0.5 * hold_count

   # Try continuous ramp starting lower
   hold_progress = max(0, cube_z - 0.04) / (lift_height - 0.04)
   reward += hold_progress * 0.5 * hold_count
   ```

2. **Earlier hold signal**: Start counting hold at lower height (e.g., 4cm instead of 8cm)

3. **Reduce hold_steps requirement**: 150 steps (3 sec) may be too long for exploration to discover

## Verification

Run training with fixed IK to verify multi-env parallel simulation works correctly:

```bash
uv run python src/training/train_drqv2.py
```

Check that all 16 envs show independent gripper-to-cube distances in the first few episodes.
