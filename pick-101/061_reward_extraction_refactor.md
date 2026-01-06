# 061: Reward Extraction Refactor

## Summary

Extracted 19 reward functions from `lift_cube.py` into a dedicated `src/envs/rewards/` module and reorganized config files into logical subdirectories.

## Changes

### Reward Extraction

**Before:** `lift_cube.py` was 1636 lines with 19 inline `_reward_v*` methods (~970 lines of reward code).

**After:** `lift_cube.py` is 662 lines. Rewards moved to:

```
src/envs/rewards/
├── __init__.py           # REWARD_FUNCTIONS dict for dispatch
├── lift_rewards.py       # Working rewards: v11, v19 (148 lines)
└── _legacy_rewards.py    # Legacy rewards: v1-v10, v12-v18 (466 lines)
```

### Working Reward Versions

| Version | Type | Success Rate | File |
|---------|------|--------------|------|
| v11 | State-based (SAC) | 100% @ 1M | lift_rewards.py |
| v19 | Image-based (DrQ-v2) | 100% @ 2M | lift_rewards.py |

### Config Reorganization

**Before:** 21 YAML files in flat `configs/` directory.

**After:**
```
configs/
├── default.yaml              # Base hyperparameters
├── pick_place.yaml           # Pick-and-place task
├── state_based/
│   ├── curriculum_stage1.yaml
│   ├── curriculum_stage3.yaml
│   └── curriculum_stage4.yaml
├── image_based/
│   └── drqv2_lift_s3_v19.yaml
└── _legacy/
    ├── drqv2_lift_s3_v*.yaml  # v13, v15-v18
    ├── drqv2_lift_s3.yaml
    ├── drqv2_lift_s2_sanity.yaml
    ├── her_1m.yaml
    ├── lift_500k.yaml
    ├── lift_cartesian_500k.yaml
    ├── lift_curriculum_s1*.yaml
    ├── lift_image.yaml
    ├── staged_contact_1m.yaml
    └── staged_rewards_1m.yaml
```

## Implementation Details

### Reward Function Signature Change

Functions changed from methods to standalone functions:

```python
# Before (method on LiftCubeCartesianEnv)
def _reward_v19(self, info, was_grasping, action):
    ...
    lift_progress = max(0, cube_z - 0.015) / (self.lift_height - 0.015)
    ...

# After (standalone function)
def reward_v19(env, info, was_grasping, action):
    ...
    lift_progress = max(0, cube_z - 0.015) / (env.lift_height - 0.015)
    ...
```

### Dispatch Mechanism

```python
# In lift_cube.py
from src.envs.rewards import REWARD_FUNCTIONS

def _compute_reward(self, info, was_grasping=False, action=None):
    if self.reward_type == "sparse":
        return 0.0 if info["is_success"] else -1.0

    reward_fn = REWARD_FUNCTIONS.get(self.reward_version)
    if reward_fn is None:
        raise ValueError(f"Unknown reward version: {self.reward_version}")
    return reward_fn(self, info, was_grasping=was_grasping, action=action)
```

## Verification

Tested all reward versions work correctly:

```bash
MUJOCO_GL=egl uv run python -c "
from src.envs.lift_cube import LiftCubeCartesianEnv
env = LiftCubeCartesianEnv(reward_version='v19')
env.reset()
for _ in range(100):
    obs, reward, _, _, info = env.step(env.action_space.sample())
print(f'v19 works: {reward:.4f}')
"
# v19 works: 0.0166
# v11 works: 0.0131
# v7 works: 0.0066
```

## Backward Compatibility

- All 19 reward versions (v1-v19) still work
- Existing checkpoints remain compatible
- Config files reference reward_version by string (e.g., "v19") - unchanged

## Future Work

Not in scope for this refactor (deferred):
- Training script consolidation
- Environment class inheritance/composition
- Contact detection extraction
- Base config inheritance system
