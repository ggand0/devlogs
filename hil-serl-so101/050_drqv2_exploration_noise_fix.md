# DrQ-v2 Exploration Noise Fix

## Problem

During HIL-SERL training, the robot only drifted to the right with no sign of learning. Only in episode 1 it shook left and right (random exploration), then settled into deterministic behavior.

## Root Cause

In `actor.py`, the `select_action` call was missing exploration parameters:

```python
# Before (wrong)
action = policy.select_action(batch=obs)

# After (correct)
action = policy.select_action(batch=obs, step=interaction_step, eval_mode=False)
```

The `DrQV2Policy.select_action` method has:
- `eval_mode: bool = True` (default) - outputs deterministic mean action
- `step: int = 0` (default) - used for stddev schedule

When called without arguments:
1. `eval_mode=True` → no noise added, just `action = dist.loc`
2. `step=0` → doesn't matter since eval_mode skips noise anyway

## Fix

Pass `eval_mode=False` and current step to enable exploration:

```python
action = policy.select_action(batch=obs, step=interaction_step, eval_mode=False)
```

This enables:
1. Action sampling with noise: `action = dist.sample()`
2. Stddev schedule: `linear(0.2, 0.1, 10000)` decays noise over training

## File Modified

`/home/gota/ggando/ml/lerobot/src/lerobot/scripts/rl/actor.py` line 508
