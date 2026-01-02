# Devlog 032: Training Speedup Options for Image-Based RL

## Context

Current image-based RL training runs at ~19-20 it/s on AMD RX 7900 XTX, resulting in ~30 hours for 2M timesteps. This document explores options for faster training.

## Current Bottleneck Analysis

```
Image-based RL pipeline (per step):
┌─────────────────────┐
│ MuJoCo Physics      │ ← CPU-bound (~40%)
│ (mj_step)           │
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ Camera Rendering    │ ← CPU/OpenGL-bound (~50%)
│ (84x84 RGB)         │
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ CNN Forward/Backward│ ← GPU (~10%, NOT the bottleneck)
│ (SAC update)        │
└─────────────────────┘
```

The GPU is underutilized. The bottleneck is CPU physics + rendering.

## Option 1: More CPU Cores / Parallel Envs

**Speedup**: ~2-4x
**Effort**: Low (config change)
**AMD Compatible**: Yes

```yaml
# Current
num_train_envs: 1  # DummyVecEnv

# Could use SubprocVecEnv with more envs
num_train_envs: 8  # Needs 8+ CPU cores
```

**Limitation**: Diminishing returns after ~8-16 envs due to memory bandwidth.

## Option 2: Lower Resolution Images

**Speedup**: ~1.3x
**Effort**: Trivial
**AMD Compatible**: Yes

```yaml
image:
  size: 64  # Instead of 84
```

**Trade-off**: May reduce policy quality, but often works fine.

## Option 3: MJX (MuJoCo XLA) - State-Based Only

**Speedup**: 10-100x
**Effort**: Medium (env rewrite)
**AMD Compatible**: Yes (JAX supports ROCm)

```python
# GPU-accelerated physics
import mujoco.mjx as mjx
data = mjx.step(model, data)  # Runs on GPU
```

**Limitation**: MJX does NOT support rendering. Only works for state-based RL.

## Option 4: IsaacGym/IsaacLab (NVIDIA Only)

**Speedup**: 50-100x
**Effort**: High (full port)
**AMD Compatible**: NO - NVIDIA GPUs only

```python
# Full GPU pipeline: physics + rendering
gym = gymapi.acquire_gym()
sim_params.use_gpu_pipeline = True
# 10,000-50,000 FPS possible
```

**Key advantage**: GPU-accelerated rendering for image-based RL.

### IsaacGym Performance Examples

| Task | CPU MuJoCo | IsaacGym |
|------|------------|----------|
| Ant locomotion | ~100 FPS | ~50,000 FPS |
| Shadow Hand | ~50 FPS | ~20,000 FPS |
| Lift task (est.) | ~20 FPS | ~10,000-30,000 FPS |

## Option 5: Distributed RL (RLlib, Sample Factory)

**Speedup**: 2-10x
**Effort**: High (framework migration)
**AMD Compatible**: Yes

Scales across multiple machines/GPUs but requires significant code changes.

## Recommendation Summary

| Goal | Best Option | Hardware Needed |
|------|-------------|-----------------|
| Quick win | Lower resolution + more envs | Current |
| State-based speedup | MJX | Current (JAX) |
| Image-based speedup | IsaacGym | NVIDIA GPU |
| Parallel experiments | Second GPU | Any |

## Hardware Consideration

For a GPU server build:

**Option A: 2x AMD 7900 XTX**
- Parallel experiments only
- No IsaacGym access
- ROCm ecosystem limitations

**Option B: AMD 7900 XTX + NVIDIA 3090**
- Regular MuJoCo on AMD
- IsaacGym experiments on NVIDIA
- Best of both ecosystems
- 3090 used market: ~$700-900

The hybrid setup provides more capabilities than 2x AMD.

## Conclusion

For image-based RL, the rendering bottleneck limits speedup options on AMD. The practical paths are:

1. **Short term**: Accept ~30hr training, run parallel experiments
2. **Medium term**: Add NVIDIA GPU for IsaacGym access
3. **Long term**: Full IsaacGym port for 50-100x speedup

The 30-hour training becomes ~30 minutes with IsaacGym.
