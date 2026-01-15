# Devlog 021: PPO 1M Step Results & DrQ-v2 Switch

**Date:** 2026-01-12

## PPO Training Results (1M Steps)

Trained PPO with the multi-env wrist camera setup for 1M timesteps.

### Configuration
- Run: `runs/ppo_multi_env_wrist/20260112_002549`
- Checkpoint: `checkpoint_1001472.pt`
- Environments: 16 parallel envs
- Image input: Wrist camera (84x84 RGB)
- Domain randomization: Enabled

### Evaluation Results
```
Checkpoint: checkpoint_1001472.pt
Episodes: 3
Mean Reward: 28.80 +/- 20.14
Success Rate: 0.0%
Max Cube Z: 0.018 +/- 0.004
Mean Grasp Steps: 0
```

### Observed Behavior

The agent exhibits consistent failure patterns across all evaluation videos:

1. **Leftward Drift**: The arm consistently drifts to the left (-Y direction in robot frame) regardless of cube position
2. **Static Finger Hooking**: Sometimes hooks the cube with the static/fixed finger while drifting, but never achieves a proper grasp
3. **No Lifting**: Max cube height ~2cm (just incidental contact, not intentional lifting)
4. **Reward Stagnation**: Mean reward ~28-56 across checkpoints (150k-1M), no clear upward trend

### Analysis

PPO is struggling with this task for several reasons:

1. **Sample Inefficiency**: Vision-based RL with PPO requires millions of samples. 1M steps is likely insufficient for a 6-DoF manipulation task with image observations
2. **Exploration**: PPO's on-policy nature limits exploration efficiency. The leftward drift suggests the policy collapsed to a local minimum
3. **Sparse Reward Signal**: Despite reward shaping, the task fundamentally requires precise sequential behaviors (approach, align, grasp, lift, hold) that are hard to discover through random exploration

### Decision: Switch to DrQ-v2

DrQ-v2 (Data-regularized Q-learning v2) is better suited for this task:

| Aspect | PPO | DrQ-v2 |
|--------|-----|--------|
| Sample efficiency | ~10M+ steps typical | ~100k-500k steps |
| Off-policy | No | Yes (replay buffer) |
| Image augmentation | Manual | Built-in (shift, crop) |
| Exploration | Policy entropy | SAC-style + augmentation |
| Proven domains | Game-like, locomotion | DMControl, manipulation |

DrQ-v2 key features:
- **Data augmentation**: Random shifts provide implicit regularization and improve generalization
- **Off-policy learning**: Replay buffer enables learning from past experiences
- **Actor-critic with SAC**: Better exploration through entropy regularization
- **n-step returns**: Faster credit assignment for delayed rewards

## DrQ-v2 Implementation

### Native Multi-Env for RoboBase

Implemented native Genesis multi-env for DrQ-v2 training using the same efficient approach as PPO (9.6x speedup at 16 envs).

**Key Challenge**: RoboBase wrappers (`FrameStack`, `ConcatDim`, `RescaleFromTanh`, `AppendDemoInfo`) inherit from `gym.Wrapper`, which asserts `isinstance(env, gym.Env)`. However, `gym.vector.VectorEnv` is NOT a subclass of `gym.Env`, so wrapping fails.

**Solution**: Created `WrappedGenesisVectorEnv` that integrates all transforms directly instead of using wrapper chain.

### WrappedGenesisVectorEnv

Location: `src/training/genesis_factory.py`

Integrates the following transforms:
- **SuccessInfo**: `is_success` → `task_success` for RoboBase compatibility
- **RescaleFromTanh**: Actions from [-1, 1] to original space
- **FrameStack**: Stack observations over time (axis=1 for batch dimension)
- **ActionSequenceMask**: Add `action_sequence_mask` to info (always 1 for RL)
- **AppendDemoInfo**: Add `demo` flag to info

```python
class WrappedGenesisVectorEnv(gym.vector.VectorEnv):
    def __init__(
        self,
        num_envs: int,
        curriculum_stage: int,
        reward_version: str,
        image_size: int,
        max_episode_steps: int,
        frame_stack: int,
    ):
        # No default values - all from config
        ...
```

### VectorEnv Initialization Fix

`gym.vector.VectorEnv.__init__()` doesn't accept parameters. Must set attributes directly:

```python
class GenesisVectorEnv(gym.vector.VectorEnv):
    def __init__(self, num_envs, ...):
        super().__init__()  # No parameters!

        # Set VectorEnv attributes directly
        self.num_envs = num_envs
        self.is_vector_env = True
        self.single_observation_space = gym.spaces.Dict({...})
        self.single_action_space = gym.spaces.Box(...)

        from gymnasium.vector.utils import batch_space
        self.observation_space = batch_space(self.single_observation_space, num_envs)
        self.action_space = batch_space(self.single_action_space, num_envs)
```

### Config-Driven Parameters

All parameters come from `src/training/cfgs/genesis_lift.yaml` - no hard-coded defaults:

```yaml
env:
  env_name: genesis_lift
  episode_length: 200
  image_size: 84
  curriculum_stage: 3
  reward_version: v11
  stddev_schedule: "linear(1.0,0.1,500000)"

pixels: true
frame_stack: 3
num_train_envs: 8
batch_size: 256
```

Factory uses direct config access:
```python
def make_train_env(self, cfg: DictConfig) -> gym.vector.VectorEnv:
    return WrappedGenesisVectorEnv(
        num_envs=cfg.num_train_envs,
        curriculum_stage=cfg.env.curriculum_stage,
        reward_version=cfg.env.reward_version,
        image_size=cfg.env.image_size,
        max_episode_steps=cfg.env.episode_length,
        frame_stack=cfg.frame_stack,
    )
```

### Observation Format

After frame stacking:
- `rgb`: `(N, frame_stack, C, H, W)` = `(8, 3, 3, 84, 84)`
- `low_dim_state`: `(N, frame_stack, 18)` = `(8, 3, 18)`

### Training Command

```bash
uv run python src/training/train_drqv2.py --config src/training/cfgs/genesis_lift.yaml
```

Resume from checkpoint:
```bash
uv run python src/training/train_drqv2.py \
    --config src/training/cfgs/genesis_lift.yaml \
    --resume runs/genesis_rl/TIMESTAMP/snapshots/latest_snapshot.pt
```

### Benchmark

With 8 envs: 67.9 samples/sec (8.5x vs single env)

### Files Modified

- `src/training/genesis_factory.py`: Added `WrappedGenesisVectorEnv`, updated `make_train_env`
- `src/training/genesis_gym_wrapper.py`: Fixed `GenesisVectorEnv` initialization
- `src/training/cfgs/genesis_lift.yaml`: Config for DrQ-v2 training

## Bug Fix: Wrong Reward Version (v11 vs v19)

### Problem

DrQ-v2 training regressed after 200k steps - deterministic eval policy moved away from cube while stochastic exploration still got rewards. Evaluated 500k checkpoint: mean reward 16 (vs 216 at 200k), 0% success.

### Root Cause

Genesis was using `reward_version: v11` (designed for **state-based SAC**) instead of `reward_version: v19` (designed for **image-based DrQ-v2**).

| Metric | v11 (wrong) | v19 (correct) |
|--------|-------------|---------------|
| Grasp bonus | 0.25 | **1.5** (6x stronger) |
| Lift coefficient | 2.0 | **4.0** (2x stronger) |
| Hold count bonus | None | **0.5 × hold_count** |
| Per-finger reach | No | Yes |

v19 achieved 100% success at 800k steps in MuJoCo with ±2cm cube position DR.

### Why v19 Wasn't Working in Genesis

v19 calls `env._get_moving_finger_pos()` for per-finger reach reward, but Genesis didn't have this method.

### Fix

1. Added `_get_moving_finger_pos()` to Genesis env (`src/envs/lift_cube_env.py`)
2. Added `moving_finger_pos` to info dict for per-env reward computation
3. Updated v19 to use info dict (works for both MuJoCo and Genesis)
4. Changed config from `reward_version: v11` to `reward_version: v19`

Commit: `7ab9923` - Switch to v19 reward for image-based DrQ-v2 training

### Additional Fix: Rollout Success Rate Logging

Success rate was always showing 0% even when episodes succeeded. Cause: `task_success` was only added to top-level `info` dict, not to `final_info` entries used by the workspace for rollout logging.

Commit: `7939690` - Fix rollout success rate logging for multi-env training

### References

- [DrQ-v2 Paper](https://arxiv.org/abs/2107.09645) - Mastering Visual Continuous Control: Improved Data-Augmented Reinforcement Learning
- [DrQ-v2 GitHub](https://github.com/facebookresearch/drqv2)
