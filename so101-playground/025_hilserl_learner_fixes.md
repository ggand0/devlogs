# Devlog 025: HIL-SERL Learner Fixes

## Overview

Fixed multiple issues preventing HIL-SERL training from starting with the SO-101 robot and teleoperation dataset.

---

## Fix 1: Dataset Key Mismatch

### Problem

The learner crashed when loading the offline replay buffer:
```
KeyError: 'observation.image'
```

The policy's `input_features` specified `observation.image`, but the dataset has `observation.images.gripper_cam`.

### Root Cause

The `ReplayBuffer.from_lerobot_dataset()` method uses `state_keys=cfg.policy.input_features.keys()` to access dataset samples directly. The `features_map` in the env config is only used at runtime by the environment wrapper, not when loading offline data.

### Solution

Changed policy config to use the dataset's actual key names:

```json
"input_features": {
    "observation.images.gripper_cam": {
        "type": "VISUAL",
        "shape": [3, 84, 84]
    },
    "observation.state": {
        "type": "STATE",
        "shape": [18]
    }
}
```

Also updated `dataset_stats` and `features_map` to use matching keys.

### Files Changed

- `train_hilserl_drqv2.json`

---

## Fix 2: Missing Reward Key

### Problem

After fixing the observation key, learner crashed with:
```
KeyError: 'next.reward'
```

Teleoperation datasets don't have reward annotations - rewards are computed by the reward classifier at runtime during HIL-SERL training.

### Root Cause

`buffer.py:_lerobotdataset_to_transitions()` assumed `next.reward` exists in the dataset.

### Solution

Added check for missing `next.reward` key, similar to existing handling for `next.done`:

```python
# Check if the dataset has "next.done" and "next.reward" keys
sample = dataset[0]
has_done_key = "next.done" in sample
has_reward_key = "next.reward" in sample

# ...

if has_reward_key:
    reward = float(current_sample["next.reward"].item())
else:
    reward = 0.0  # Default for datasets without reward annotations
```

### Files Changed

- `lerobot/src/lerobot/utils/buffer.py`

---

## Fix 3: Replay Buffer Caching

### Problem

Converting the LeRobot dataset to replay buffer format takes ~8 minutes for 15858 samples. This happens on every learner startup, making iteration slow.

### Solution

Added caching to save/load the converted replay buffer:

1. **Added `save()` and `load()` methods to ReplayBuffer class:**

```python
def save(self, path: str) -> None:
    """Save the replay buffer to disk."""
    save_dict = {
        "states": self.states,
        "actions": self.actions,
        "rewards": self.rewards,
        "dones": self.dones,
        # ... other fields
    }
    torch.save(save_dict, f"{path}.pt")

@classmethod
def load(cls, path: str, device: str, ...) -> "ReplayBuffer":
    """Load a replay buffer from disk."""
    save_dict = torch.load(f"{path}.pt", weights_only=False)
    # ... restore state
```

2. **Modified `initialize_offline_replay_buffer()` in learner.py:**

```python
# Check for cached buffer
cache_path = os.path.join(cfg.output_dir, "offline_buffer")
cache_file = f"{cache_path}.pt"

if os.path.exists(cache_file):
    logging.info(f"Loading cached offline replay buffer from {cache_file}")
    return ReplayBuffer.load(cache_path, device=device, ...)

# No cache - convert and save
offline_replay_buffer = ReplayBuffer.from_lerobot_dataset(...)
offline_replay_buffer.save(cache_path)
return offline_replay_buffer
```

### Result

- First run: ~8 minutes (full conversion + cache save)
- Subsequent runs: ~seconds (direct load from cache)
- Cache location: `{output_dir}/offline_buffer.pt`
- Delete cache file to force re-conversion

### Files Changed

- `lerobot/src/lerobot/utils/buffer.py` - Added `save()` and `load()` methods
- `lerobot/src/lerobot/scripts/rl/learner.py` - Added caching logic

---

## Summary of Config Changes

The final working `train_hilserl_drqv2.json` includes:

| Field | Value | Reason |
|-------|-------|--------|
| `policy.type` | `"sac"` | HIL-SERL uses SAC, not DrQ-v2 |
| `policy.input_features` | `observation.images.gripper_cam` | Match dataset keys |
| `dataset.video_backend` | `"pyav"` | torchcodec not available |
| `policy.offline_buffer_capacity` | `50000` | Must be >= dataset length (15858) |
| `wandb.enable` | `false` | Disable wandb for local testing |

---

## Usage

```bash
# Run learner (first terminal)
uv run python -m lerobot.scripts.rl.learner --config_path train_hilserl_drqv2.json

# Run actor (second terminal) - requires robot connected
uv run python -m lerobot.scripts.rl.actor --config_path train_hilserl_drqv2.json
```

---

## Related

- Devlog 024: MuJoCo IK integration for SO101FollowerEndEffector
- `lerobot/src/lerobot/utils/buffer.py` - ReplayBuffer implementation
- `lerobot/src/lerobot/scripts/rl/learner.py` - Learner implementation
