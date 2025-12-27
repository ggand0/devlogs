# Devlog 028: CNN Training Optimizations

## Date: 2025-12-25

## Overview

Implemented four optimizations to speed up CNN-based RL training and improve learning:

1. **84x84 resolution** (default, was 256x256) - 9x smaller images
2. **Frame stacking** - Helps CNN infer velocity from consecutive frames
3. **Grayscale mode** - 3x smaller images (optional)
4. **Parallel environments** - Run N game instances for ~N× faster data collection

Combined potential reduction: **~27x smaller** images than original 256x256 RGB, plus **~2x faster** with parallel envs.

## Commits

- `d7925a3` - Add configurable image resolution and frame stacking support
- `76728d2` - Add grayscale mode for image observations

## Implementation Details

### 1. 84x84 Resolution (Atari-standard)

Changed default image dimensions from 256x256 to 84x84:

| Metric | 256x256 | 84x84 | Reduction |
|--------|---------|-------|-----------|
| Pixels | 65,536 | 7,056 | **9.3x** |
| RGB bytes | 196KB | 21KB | **9.3x** |
| Replay buffer (50K) | ~20GB | ~2.1GB | **9.3x** |

**Files changed:**
- `src/config.rs`: Changed `IMAGE_OBS_WIDTH/HEIGHT` constants from 256 to 84
- Added `image_obs_width` and `image_obs_height` to `GameConfig`
- `src/rl/image_observation.rs`: Functions now accept width/height parameters
- `src/rl/api.rs`: `/configure` endpoint accepts dimensions, `/observation_space` returns configured size
- `src/main.rs`: Pass configured dimensions to image generation

### 2. Frame Stacking

Added SB3's `VecFrameStack` wrapper to stack consecutive frames, allowing CNN to infer motion/velocity.

**Configuration:**
```yaml
frame_stack: 4  # Stack 4 consecutive frames
```

**Observation shape change:**
- Without stacking: `(84, 84, 3)` or `(84, 84, 1)` for grayscale
- With 4-frame stack: `(84, 84, 12)` or `(84, 84, 4)` for grayscale

**Files changed:**
- `python/config.py`: Added `frame_stack: Optional[int]` field
- `python/train_sac.py`: Apply `VecFrameStack` wrapper when `frame_stack > 1`

**Why frame stacking helps:**
- Single frame = no velocity information (static snapshot)
- Stacked frames = CNN can compute motion by comparing consecutive frames
- Enables prediction of projectile trajectories

### 3. Grayscale Mode

Optional single-channel output using luminance conversion.

**Luminance formula:** `Y = 0.299*R + 0.587*G + 0.114*B`

**Entity grayscale values:**

| Entity | RGB Color | Grayscale |
|--------|-----------|-----------|
| Background | (30, 30, 40) | ~31 |
| Projectiles | (255, 50, 50) | ~111 |
| Player | (50, 150, 255) | ~132 |
| Thrower | (255, 140, 0) | ~158 |
| Zone boundary | (200, 200, 50) | ~185 |

**Configuration:**
```yaml
image_grayscale: true  # Use 1 channel instead of 3
```

**Files changed:**
- `src/config.rs`: Added `image_grayscale: bool` to `GameConfig`, `image_channels()` helper
- `src/rl/image_observation.rs`: Updated `generate_synthetic_topdown_image_into()` to support grayscale
- `src/rl/api.rs`: Returns correct channel count, encodes only needed bytes
- `src/main.rs`: Handle `image_grayscale` in Configure command
- `python/config.py`: Added `image_grayscale: Optional[bool]`
- `python/bevy_dodge_env/environment.py`: Added to `configure()` method
- `python/train_sac.py`: Pass `image_grayscale` to game configuration

## Memory Comparison

| Configuration | Image Size | Buffer (50K) |
|---------------|------------|--------------|
| 256x256 RGB (original) | 196KB | ~20GB |
| 84x84 RGB | 21KB | ~2.1GB |
| 84x84 Grayscale | 7KB | ~700MB |
| 84x84 RGB + 4-frame stack | 84KB | ~8.4GB |
| 84x84 Grayscale + 4-frame stack | 28KB | ~2.8GB |

## Recommended Configuration

For next training run with all optimizations:

```yaml
# python/configs/sac_topdown_optimized.yaml
algorithm: SAC
port: 8000
level: 2

# Image observation with optimizations
observation_mode: topdown
frame_stack: 4           # Stack 4 frames for velocity inference
image_grayscale: false   # Keep RGB for now (better entity distinction)

# Arena tuning
action_space_type: basic_3d
spawn_angle_degrees: 30
sprint_multiplier: 1.0

# Training
total_timesteps: 500000
learning_rate: 0.0003
buffer_size: 100000      # Can increase now that images are smaller
learning_starts: 10000
batch_size: 128          # Can increase with smaller images
train_freq: 4
gradient_steps: 1

# Checkpointing
save_freq: 50000
eval_freq: 10000
n_eval_episodes: 5
```

## Expected Training Speed

| Configuration | Estimated FPS | Notes |
|---------------|---------------|-------|
| 256x256 RGB | ~9 FPS | Original (very slow) |
| 84x84 RGB | ~30-40 FPS | 3-5x faster |
| 84x84 Grayscale | ~40-50 FPS | Additional speedup |

## Usage

### Training with 84x84 + Frame Stacking (RGB)

```bash
# Terminal 1 - Start game server
cargo run --release -- --headless

# Terminal 2 - Start training
uv run python python/train_sac.py --config python/configs/sac_topdown_cnn.yaml
```

Add to your YAML config:
```yaml
frame_stack: 4
```

### Training with Grayscale

Add to your YAML config:
```yaml
image_grayscale: true
frame_stack: 4
```

### 4. Parallel Environments (Vectorized Training)

Run multiple Bevy game instances in parallel for N× faster data collection.

**Configuration:**
```yaml
n_envs: 2  # Number of parallel game instances (requires 2 game servers)
```

**Files changed:**
- `python/config.py`: Added `n_envs: int = 1` field
- `python/bevy_dodge_env/vec_env.py`: Updated `make_vec_env()` to accept `config_kwargs` for game configuration
- `python/bevy_dodge_env/environment.py`: Added `refresh_spaces()` method to re-query observation/action spaces after configuration
- `python/train_sac.py`: Use `SubprocVecEnv` when `n_envs > 1`, pre-configure all game servers before creating parallel environments
- `start_parallel_servers.sh`: New script to launch N headless game servers
- `python/configs/sac_topdown_cnn_parallel.yaml`: Example config with `n_envs: 2`

**Usage:**
```bash
# Terminal 1: Start N parallel game servers
./start_parallel_servers.sh 2

# Terminal 2: Train with parallel environments
uv run python python/train_sac.py --config python/configs/sac_topdown_cnn_parallel.yaml
```

**Performance results:**

| n_envs | FPS | Projected Time (500K) | Speedup |
|--------|-----|----------------------|---------|
| 1 | 15 | ~12 hours | 1x |
| 2 | 26 | ~4.5 hours | ~1.7x |

Note: Speedup is less than 2x due to SubprocVecEnv IPC overhead and GPU sharing.

## Notes

- Image dimensions and grayscale settings require server restart to resize the internal buffer
- Frame stacking is handled entirely in Python (SB3's VecFrameStack)
- The 256x256 resolution is now the default for accurate spatial representation
- Grayscale is optional and defaults to false (RGB)
- Parallel environments require starting N game servers before training

---

## Bug Fix: Inaccurate Entity Sizes in Synthetic Image (2025-12-27)

### Issue

The synthetic top-down image was using **inflated entity sizes** that didn't match the actual game:

| Entity | Game (actual) | Synthetic (was) | Error |
|--------|---------------|-----------------|-------|
| Projectile | `Sphere::new(0.3)` | 0.5 | **67% too large** |
| Player | `Capsule3d::new(0.5, 1.0)` | 0.6 | 20% too large |
| Thrower | `Sphere::new(0.2)` | 0.8 | **300% too large** |

This meant the CNN was learning incorrect spatial relationships - projectiles appeared much larger relative to the player than they actually are in the game.

### Root Cause

The original 84x84 resolution made entities too small to see clearly (projectile = ~1px diameter), so sizes were artificially inflated. But this created a mismatch between what the CNN saw and the actual game physics.

### Fix

1. **Reverted to 256x256 resolution** - Entities are now clearly visible at actual sizes:
   - Projectile (0.3 radius): ~6.4px diameter
   - Player (0.5 radius): ~10.7px diameter

2. **Updated entity radii to match game code:**
   - Projectile: 0.5 → **0.3** (matches `Sphere::new(0.3)` in projectile.rs)
   - Player: 0.6 → **0.5** (matches `Capsule3d::new(0.5, 1.0)` in player.rs)
   - Thrower: 0.8 → **0.2** (matches `Sphere::new(0.2)` in projectile.rs)

### Files Changed

- `src/config.rs`: `IMAGE_OBS_WIDTH/HEIGHT` 84 → 256
- `src/rl/image_observation.rs`: Updated `draw_circle` calls with actual entity radii

### Impact

The CNN now sees accurate spatial relationships matching game physics. This should improve learning since the model sees the same proportions it needs to reason about for collision avoidance.
