# Devlog 025: Top-Down CNN Image Observation Training

## Date: 2025-12-22 to 2025-12-23

## Objective

Train a SAC agent using 256x256 RGB top-down image observations instead of the 65-dim vector observation. This tests whether a CNN-based policy can learn to dodge projectiles purely from visual input.

## Implementation

### Observation Mode

Added `TopDownImage` mode to `ObservationMode` enum:
- Returns 256x256x3 uint8 image (RGB)
- Rendered from orthographic camera above the arena
- Base64 encoded in API responses for efficiency

### Configuration

**Config file:** `python/configs/sac_topdown_cnn_easy.yaml`
```yaml
algorithm: SAC
observation_mode: topdown  # 256x256 RGB
level: 1                   # Easy - fixed spawn direction
action_space_type: basic_3d
buffer_size: 100000        # Smaller for memory
batch_size: 64             # Smaller for GPU memory
learning_starts: 5000
total_timesteps: 500000
```

### Policy

SB3's `CnnPolicy` with NatureCNN feature extractor:
- Conv layers extract spatial features
- MLP head (`net_arch: [256, 256]`) after CNN features
- Automatic `VecTransposeImage` wrapper for channel-first format

## Training Progress

### Phase 1: Initial Learning (0-65K steps)
- Started with random policy (~20 step episodes, -80 reward)
- Gradual improvement as CNN learns to detect projectiles
- **Phase transition at ~65K**: Sudden jump to 824 eval reward

### Phase 2: Continued Training (65K-125K steps)
- Training reward showed high variance (exploration noise)
- Eval reward (deterministic) consistently higher
- Performance stabilized around 400-500 training reward
- Best eval: 857 reward at 115K steps

### Key Observations

**Training vs Eval Gap:**
- Training uses stochastic policy (exploration) → lower reward, shorter episodes
- Eval uses deterministic policy → higher reward, longer episodes
- This is expected and healthy for SAC

**Phase Transition:**
- CNN learning is non-linear
- Early training: CNN learns basic feature detection
- Sudden improvement: CNN "clicks" on projectile-player relationship
- Similar to deep learning loss curves with sudden drops

## Disk Space Management

### Problem: Replay Buffer Size

Image observations create massive replay buffers:
- Each transition: ~200KB (256x256x3 bytes)
- Buffer size 100K → ~20GB per checkpoint
- Multiple checkpoints quickly exhaust disk space

### Solution: Selective Buffer Retention

```bash
# Keep only latest replay buffer per experiment
find results -name "*.pkl" ! -name "*<latest_steps>*" -delete
```

**Recommendation:** Set `save_replay_buffer=False` in CheckpointCallback for image observations, or implement automatic cleanup of old buffers.

### Checkpoint File Sizes

| File Type | Size | Notes |
|-----------|------|-------|
| Model `.zip` | ~700MB | Policy weights, optimizer state |
| Replay buffer `.pkl` | 17-39GB | Depends on buffer fill level |

## Resume Functionality

### Improvements Made

Modified `train_sac.py` to:
1. Accept direct `.zip` file paths for resume
2. Prioritize latest checkpoint over `final_model.zip`
3. Auto-discover checkpoints from run directory

```python
# Resume options:
--resume results/sac_topdown_cnn/20251222_185151  # Auto-select latest
--resume path/to/specific_model.zip               # Exact file
```

### Buffer Behavior on Resume

- Loading `.zip` only restores policy weights
- Replay buffer starts **empty** unless `.pkl` explicitly loaded
- 5K step warmup before training resumes (`learning_starts`)
- Buffer refills naturally during training

## Results Summary

| Metric | Value |
|--------|-------|
| Best Eval Reward | 857 (at 115K steps) |
| Best Eval Episode Length | 873 steps |
| Training Reward (converged) | 400-500 |
| FPS | ~8-9 it/s |
| Total Training Time | ~14 hours for 125K steps |

## Files Modified

| File | Changes |
|------|---------|
| `src/config.rs` | Added `TopDownImage` to `ObservationMode` |
| `src/rl/observation.rs` | Handle image mode in extraction |
| `src/rl/api.rs` | Base64 image encoding, observation space shape |
| `python/train_sac.py` | Resume logic, CnnPolicy detection, VecTransposeImage |
| `python/bevy_dodge_env/environment.py` | Image decoding, `is_image_observation` property |
| `python/configs/sac_topdown_cnn_easy.yaml` | New config for CNN training |

## Lessons Learned

1. **CNN training is compute-intensive**: ~8 it/s vs ~100+ for vector obs
2. **Replay buffers dominate disk**: 20GB+ per checkpoint with images
3. **Phase transitions are normal**: Expect non-linear learning curves
4. **Train from latest, deploy from best**: Keep both checkpoints
5. **Buffer not required for resume**: Model weights are sufficient, buffer rebuilds

## Next Steps

1. Continue training to 500K steps
2. Compare final performance vs vector observation baseline
3. Test on Level 2 (harder, random spawn angles)
4. Consider frame stacking for temporal information
5. Experiment with smaller image sizes (128x128) for faster training
