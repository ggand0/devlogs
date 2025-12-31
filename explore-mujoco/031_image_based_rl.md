# Devlog 031: Image-Based RL with RoboBase

## Overview

Implementation of image-based reinforcement learning for SO-101 lift task using the RoboBase framework with DrQ-v2 algorithm. This enables sim-to-real transfer using only wrist camera observations.

## Motivation

State-based RL (stages 1-4) achieved good results but relies on privileged information:
- Exact cube position from simulation
- Perfect gripper position/velocity
- No sensor noise

Image-based RL learns directly from camera pixels, making the policy transferable to real hardware where only camera observations are available.

## Implementation

### Wrist Camera Addition (`resources/so101/so101_cube.xml`)

Added a camera mounted on the gripper wrist:

```xml
<camera name="wrist_cam" pos="0 0 0.05" euler="180 0 -90" fovy="60"/>
```

- Position: 5cm along gripper Z-axis (forward from wrist)
- Orientation: Looking down at workspace
- FOV: 60 degrees

### Image Observation Wrapper (`src/envs/wrappers/image_obs.py`)

Wrapper that adds camera image to observations:

```python
class ImageObsWrapper(gym.ObservationWrapper):
    def __init__(self, env, image_size=(84, 84), camera="wrist_cam"):
        # Setup MuJoCo renderer
        self._renderer = mujoco.Renderer(model, height, width)

    def observation(self, obs):
        self._renderer.update_scene(self.unwrapped.data, camera=self.camera)
        img = self._renderer.render()
        return {"rgb": img, "low_dim_state": obs}
```

### RoboBase Factory (`src/training/so101_factory.py`)

Environment factory implementing RoboBase's `EnvFactory` interface:

```python
class SO101Factory(EnvFactory):
    def _wrap_env(self, env, cfg):
        env = RescaleFromTanh(env)           # Scale actions
        env = WristCameraWrapper(env, ...)   # Add images
        env = ConcatDim(env, ...)            # Concatenate state
        env = TimeLimit(env, ...)            # Episode length
        env = ActionSequence(env, ...)       # Action chunking
        env = FrameStack(env, ...)           # Stack 3 frames
        env = AppendDemoInfo(env)            # Demo info
        return env

    def make_train_env(self, cfg):
        return gym.vector.AsyncVectorEnv(
            [...], context="spawn"  # Required for EGL
        )
```

Key details:
- Uses `MUJOCO_GL=egl` for headless GPU rendering
- `multiprocessing.set_start_method("spawn")` for AMD GPU compatibility
- Image format: (C, H, W) = (3, 84, 84) RGB
- Frame stack: 3 frames for temporal information

### Hydra Configuration (`src/training/cfgs/so101_lift.yaml`)

```yaml
defaults:
  - /robobase_config
  - override /method: drqv2
  - _self_

env:
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

### Training Script (`src/training/train_image_rl.py`)

```python
# Must be first - spawn required for EGL multiprocessing
import multiprocessing
multiprocessing.set_start_method("spawn", force=True)

@hydra.main(config_path="cfgs", config_name="so101_lift")
def main(cfg):
    workspace = Workspace(cfg, env_factory=SO101Factory())
    workspace.train()
```

## Dependencies

Updated `pyproject.toml`:
- `gymnasium @ git+https://github.com/stepjam/Gymnasium.git@0.29.2` (RoboBase fork)
- `hydra-core>=1.3.0`
- `omegaconf>=2.3.0`
- `numpy<2.0.0` (RoboBase compatibility)

RoboBase installed from `/tmp/robobase_check`.

## Usage

```bash
# Run training with EGL backend (required for headless GPU rendering)
MUJOCO_GL=egl uv run python src/training/train_image_rl.py \
  num_train_envs=8 \
  wandb.use=false

# Customize training
MUJOCO_GL=egl uv run python src/training/train_image_rl.py \
  num_train_envs=8 \
  num_train_frames=2000000 \
  eval_every_steps=10000 \
  wandb.use=true wandb.name=drqv2_lift
```

## DrQ-v2 Algorithm

DrQ-v2 (Data-regularized Q-learning v2) is an image-based RL algorithm that:
1. Uses random shifts data augmentation on images
2. Applies n-step returns for better sample efficiency
3. Uses a smaller CNN encoder for faster training
4. Employs scheduled exploration noise

Key hyperparameters (from RoboBase defaults):
- Learning rate: 1e-4
- Batch size: 256
- Replay buffer: 100k transitions
- N-step returns: 3
- Gamma: 0.99
- Exploration noise: linear(1.0, 0.1, 500k)

## Initial Results

First evaluation (before any training):
```
| eval | Iter: 0 | S: 0 | E: 0 | L: 200 | R: 37.87 | T: 0:00:15
```

Training metrics after 8000 steps:
```
| train | Iter: 1000 | S: 8000 | E: 40 | BS: 8000 | Env FPS: 1229 | Update FPS: 34.5
```

## Known Issues

1. **Gymnasium warnings**: Deprecation warnings about `is_vector_env` - harmless
2. **Leaked semaphores**: Resource tracker warning at shutdown - harmless
3. **moviepy import**: Missing editor module - video recording disabled

## SB3-Based Alternative (train_lift_image.py)

Added familiar SB3-based training with CnnPolicy for users who prefer SB3's logging format.

### Usage

```bash
MUJOCO_GL=egl uv run python train_lift_image.py --config configs/lift_image.yaml
```

### Output Format

```
---------------------------------
| rollout/           |          |
|    success_rate    | 0.37     |
| time/              |          |
|    episodes        | 100      |
|    fps             | 383      |
|    time_elapsed    | 52       |
|    total_timesteps | 20000    |
| train/             |          |
|    actor_loss      | -28.4    |
|    critic_loss     | 2.24     |
---------------------------------
 100% ━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 1,000,000/1,000,000  [ 1:30:00 < 0:00:00 , 185 it/s ]
```

### Config (configs/lift_image.yaml)

```yaml
image:
  size: 84
  camera: wrist_cam
  frame_stack: 3

sac:
  learning_rate: 0.0003
  buffer_size: 100000
  batch_size: 128
```

## Comparison: RoboBase vs SB3

| Feature | RoboBase (DrQ-v2) | SB3 (SAC+CNN) |
|---------|-------------------|---------------|
| Algorithm | DrQ-v2 | SAC with CnnPolicy |
| Data augmentation | Random shifts | None |
| Logging | Custom format | Table + tqdm |
| Multi-env | AsyncVectorEnv | DummyVecEnv |
| Frame stack | RoboBase wrapper | VecFrameStack |

## Next Steps

1. Run full training to convergence (both versions)
2. Compare with state-based RL performance
3. Test sim-to-real transfer

## Files Changed

- `resources/so101/so101_cube.xml` - Added wrist camera
- `src/envs/wrappers/__init__.py` - Wrapper exports
- `src/envs/wrappers/image_obs.py` - Image observation wrapper
- `src/training/__init__.py` - Training module
- `src/training/so101_factory.py` - RoboBase environment factory
- `src/training/cfgs/so101_lift.yaml` - Hydra configuration
- `src/training/train_image_rl.py` - RoboBase training entry point
- `train_lift_image.py` - SB3-based training script
- `configs/lift_image.yaml` - SB3 training configuration
- `pyproject.toml` - Added RoboBase dependencies
