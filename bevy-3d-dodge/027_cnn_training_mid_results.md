# Devlog 027: CNN Training Mid-Run Results & Optimization Plan

## Date: 2025-12-23

## Training Status

Training paused at **~350K steps** (70% of 500K target) after ~10.5 hours of training.

### Configuration

```yaml
algorithm: SAC
observation_mode: topdown        # 256x256 RGB synthetic image
action_space_type: basic_3d      # [vx, vy, sprint]
spawn_angle_degrees: 30          # +/-30 degrees (60 total fan)
buffer_size: 50000               # ~20GB replay buffer
level: 2                         # Hard difficulty
```

### Results at 340K Steps

| Metric | Training (rollout) | Evaluation |
|--------|-------------------|------------|
| Episode Length | 38-39 steps | **282 steps** |
| Reward | -61 to -62 | **+184** |
| Variance | - | +/- 159 |

**Key Observations:**

1. **Evaluation vs Training gap is normal** - SAC uses stochastic actions during training (exploration), deterministic during eval
2. **High variance** (+/- 159 reward, +/- 157 episode length) - Agent performance is inconsistent
3. **Slow training speed** - Only 9 FPS due to large image observations
4. **Training time** - ~10.5 hours for 340K steps, ~15 hours projected for 500K

### Training Progression

| Timesteps | Eval Reward | Eval Length | Notes |
|-----------|-------------|-------------|-------|
| 270K | 195 +/- 215 | 291 | High variance |
| 340K | 184 +/- 159 | 282 | Similar performance, slightly lower variance |

## Analysis

### Why High Variance?

The agent shows inconsistent performance because **the image observation lacks velocity information**:

1. **Single frame = no motion** - A static 256x256 image shows positions but not velocities
2. **Can't predict trajectories** - Without knowing ball velocity, agent can't anticipate where balls will go
3. **Reactive only** - Agent can only react to current positions, not plan ahead

This is a fundamental limitation of single-frame image observations. Solutions:

- **Frame stacking** - Stack 3-4 consecutive frames so CNN can infer motion
- **Optical flow** - Compute velocity field as additional channel
- **Hybrid observation** - Image + velocity vector overlay

### Why Slow Training?

| Factor | Impact |
|--------|--------|
| Image size: 256x256x3 = 196KB | Large data per step |
| Base64 encoding overhead | ~262KB per HTTP response |
| CNN forward/backward pass | GPU compute time |
| Single environment | No parallelization |

Current throughput: **9 steps/second** (vs 50+ for MLP with vector obs)

## Optimization Plan

### Priority 1: Reduce Image Resolution (Biggest Impact)

Change from 256x256 to **84x84** (Atari standard):

| Metric | 256x256 | 84x84 | Reduction |
|--------|---------|-------|-----------|
| Pixels | 65,536 | 7,056 | **9.3x smaller** |
| RGB bytes | 196KB | 21KB | **9.3x smaller** |
| Base64 size | 262KB | 28KB | **9.3x smaller** |
| Buffer (50K) | ~20GB | ~2.1GB | **9.3x smaller** |

**Implementation:**
1. Make image dimensions configurable in `config.rs`
2. Update `image_observation.rs` to use config values
3. Update Python env to handle variable sizes
4. Update YAML config to specify resolution

### Priority 2: Grayscale Mode (Optional)

Convert RGB to grayscale:

| Metric | RGB | Grayscale | Reduction |
|--------|-----|-----------|-----------|
| Channels | 3 | 1 | **3x smaller** |
| 84x84 bytes | 21KB | 7KB | **3x smaller** |

For this game, color provides useful info (blue=player, red=balls, orange=thrower), but intensity differences might be sufficient.

### Priority 3: Frame Stacking

Stack 3-4 consecutive frames to provide velocity information:

```python
# SB3 built-in frame stacking
from stable_baselines3.common.vec_env import VecFrameStack
env = VecFrameStack(env, n_stack=4)
```

This would change observation from (84, 84, 3) to (84, 84, 12) but provide motion information.

### Priority 4: Vectorized Environments

Run multiple game instances in parallel:

```python
from stable_baselines3.common.vec_env import SubprocVecEnv

def make_env(port):
    def _init():
        return BevyDodgeEnv(port=port)
    return _init

# Run 4 game servers on ports 8000-8003
env = SubprocVecEnv([make_env(8000 + i) for i in range(4)])
```

**Expected speedup:** ~4x with 4 environments

**Challenges:**
- Need to run multiple Bevy game instances
- Memory usage multiplied
- Port management

### Priority 5: Training Parameter Tweaks

```yaml
# Reduce update frequency (minor speedup)
train_freq: 8           # Currently 4
gradient_steps: 2       # Currently 1
batch_size: 128         # Currently 64 (if GPU memory allows)
```

## Estimated Impact

| Optimization | Speedup | Effort |
|--------------|---------|--------|
| 84x84 resolution | ~3-5x | Medium |
| Grayscale | ~1.5x | Low |
| Frame stacking | 0.8x (slower but better learning) | Low |
| 4 parallel envs | ~3-4x | High |
| Parameter tweaks | ~1.2x | Low |

**Combined (84x84 + grayscale + 4 envs):** Could achieve **15-30x speedup**

## Next Steps

1. **Complete current run** - Let it finish to 350K for baseline comparison
2. **Implement 84x84 resolution** - Biggest bang for buck
3. **Test frame stacking** - Address velocity inference problem
4. **Consider vectorized envs** - If more speed needed

## Files to Modify for 84x84 Resolution

| File | Change |
|------|--------|
| `src/config.rs` | Add `image_obs_width`/`image_obs_height` to GameConfig |
| `src/rl/image_observation.rs` | Use config values instead of constants |
| `src/rl/api.rs` | Pass config to image buffer allocation |
| `src/main.rs` | Pass config to image generation |
| `python/configs/sac_topdown_cnn.yaml` | Add `image_width: 84`, `image_height: 84` |
| `python/bevy_dodge_env/environment.py` | Handle variable image sizes |

## Checkpoint Location

```
results/sac_topdown_cnn/20251223_125520/
├── models/
│   ├── checkpoints/
│   │   └── sac_dodge_replay_buffer_350000_steps.pkl  # ~19GB
│   └── best/
│       └── best_model.zip
└── logs/
    └── SAC_1/
        └── events.out.tfevents.*
```

Resume training with:
```bash
uv run python python/train_sac.py \
    --config python/configs/sac_topdown_cnn.yaml \
    --resume results/sac_topdown_cnn/20251223_125520/models/checkpoints/sac_dodge_350000_steps.zip
```
