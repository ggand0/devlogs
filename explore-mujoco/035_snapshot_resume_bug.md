# Devlog 035: Snapshot Resume Config Overwrite Bug

## Issue

When resuming training from a checkpoint, periodic snapshots were not being saved at the expected intervals. With `snapshot_every_n: 6250` in the config (saving every ~50k steps), no snapshots were saved between 400k and 480k steps.

## Root Cause

The `load_snapshot()` function in `robobase/workspace.py` was overwriting `self.cfg` with the config saved in the checkpoint:

```python
for k, v in payload.items():
    self.__dict__[k] = v  # This overwrites self.cfg!
```

The checkpoint was saved with the OLD config that had `snapshot_every_n: 50000`. When resuming:
- YAML config: `snapshot_every_n: 6250` (save every ~50k steps)
- Snapshot config: `snapshot_every_n: 50000` (save every ~400k steps)

The snapshot's old config value was being used at runtime, causing the next snapshot to be scheduled at 800k steps instead of 450k steps.

## Fix

Modified `load_snapshot()` to NOT restore the cfg from the snapshot:

```python
payload.pop("cfg", None)  # Don't overwrite current run's config
for k, v in payload.items():
    self.__dict__[k] = v
```

Only state variables are restored from the snapshot:
- `_main_loop_iterations`
- `_global_env_episode`
- `_pretrain_step`

The current run's Hydra config now takes precedence for all settings.

## Other Notes

### Best Model Saving
Best model saving IS working correctly. The confusion was between:
- **Training reward** (yellow logs): Agent 0's episodic reward during training
- **Eval reward** (green logs): Average reward over dedicated eval episodes

Best model is saved based on EVAL reward, not training reward.

### Eval Frequency
With `eval_every_steps: 10000`, eval runs at iterations 50000, 60000, 70000, etc.

### Commits
- `9d5fff8` (robobase): Fix resume overwriting current config with snapshot config
- `682051d` (main): Update robobase submodule
