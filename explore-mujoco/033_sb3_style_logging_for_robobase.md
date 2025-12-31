# Devlog 033: SB3-Style Logging for RoboBase DrQ-v2

## Problem

RoboBase's default logging format:
```
| eval           | Iter: 0 | S: 0 | E: 0 | L: 200 | R: 37.87 | T: 0:00:15
| train          | Iter: 1000 | S: 8000 | E: 40 | BS: 8000 | Env FPS: 1229 | Update FPS: 34.5
```

Desired SB3-style format:
```
-----------------------------------------
| rollout/                |             |
|    ep_len_mean          | 200         |
|    ep_rew_mean          | 37.87       |
| time/                   |             |
|    fps                  | 1229        |
|    iterations           | 1000        |
|    total_timesteps      | 8000        |
| train/                  |             |
|    actor_loss           | -28.4       |
|    critic_loss          | 2.24        |
-----------------------------------------
```

## Previous Mistake

Created `train_lift_image.py` using SB3's SAC+CnnPolicy instead of just modifying RoboBase's logging. This was wrong because:

1. **Missing data augmentation**: DrQ-v2 uses random shift augmentation, critical for stable image-based RL
2. **Different algorithm**: SAC vs DrQ-v2 have different exploration strategies
3. **Training instability**: SB3 version showed critic loss spikes up to 17,400 and reward regression

The user explicitly wanted only the **logging style** changed, not the training algorithm.

## Solution: Custom Logger Subclass

Instead of forking RoboBase, we create a custom Logger that:
1. Subclasses RoboBase's Logger
2. Overrides `MetersGroup._dump_to_console` for SB3-style output
3. Monkey-patches the logger before Workspace initialization

### Implementation

Create `src/training/sb3_style_logger.py`:
- Custom `SB3StyleMetersGroup` with table-style output
- `SB3StyleLogger` that uses custom meters groups
- `patch_robobase_logger()` function to apply the patch

### Usage

```python
# In train_image_rl.py, BEFORE importing Workspace
from src.training.sb3_style_logger import patch_robobase_logger
patch_robobase_logger()

# Now import and use Workspace normally
from robobase.workspace import Workspace
```

## Why Not Fork RoboBase?

1. **Maintenance burden**: Would need to track upstream changes
2. **Single feature**: Only changing logging format, not core functionality
3. **Clean separation**: Monkey-patching keeps our customization isolated
4. **Easy to revert**: Can remove the patch if RoboBase adds this feature

## Files Changed

- `src/training/sb3_style_logger.py` - Custom SB3-style logger
- `src/training/train_image_rl.py` - Apply logger patch before training
