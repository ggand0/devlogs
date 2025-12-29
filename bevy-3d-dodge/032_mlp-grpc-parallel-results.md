# MLP + gRPC + Parallel Servers Experiment Results

## Experiment Overview

**Objective**: Test MLP policy training with gRPC transport and parallel game server instances.

**Date**: December 29, 2024

**Duration**: ~1 hour 9 minutes (vs 22+ hours for CNN)

---

## Configuration

```yaml
algorithm: SAC
transport: grpc
socket_path: /tmp/bevy_rl.sock
n_envs: 4                         # 4 parallel game instances

observation_mode: with_thrower    # 69-dim vector
level: 2                          # Hard difficulty
action_space_type: basic_3d       # Continuous [vx, vy, sprint]

total_timesteps: 1000000          # 1M steps
learning_rate: 0.0003
buffer_size: 100000
batch_size: 256
train_freq: 1
gradient_steps: 1
net_arch: [256, 256]
```

**Infrastructure**:
- 4 headless Bevy instances via `start_grpc_servers.sh`
- Unix domain sockets: `/tmp/bevy_rl_0.sock` to `_3.sock`
- SubprocVecEnv for parallel data collection

---

## Results

### Final Metrics

| Metric | Value |
|--------|-------|
| Total timesteps | 1,000,000 |
| Training time | 1h 9m |
| Final eval reward | 660.48 ± 307.44 |
| Best eval reward | 874.66 |
| Final episode length | 715 steps |
| Peak episode length | 435 (training) |

### Training Throughput

| Phase | FPS |
|-------|-----|
| Before learning_starts (10K) | ~850 |
| During training | ~240-250 |
| Effective throughput | ~243 it/s average |

### Comparison to CNN

| Metric | CNN (22h) | MLP (1h) |
|--------|-----------|----------|
| Training time | 22h 42m | 1h 9m |
| Best eval reward | 164.35 | 874.66 |
| Final eval reward | 89.71 | 660.48 |
| Episode length | 186 | 715 |

**MLP achieved 5x better reward in 1/20th the time.**

---

## Visual Evaluation

The MLP agent displays clearly learned dodging behavior:

1. **Smooth, purposeful movement**: Unlike CNN's jittery panic, MLP moves deliberately
2. **Predictive dodging**: Agent anticipates ball trajectories using velocity vectors
3. **Active avoidance**: Stays mobile rather than corner-camping
4. **Occasional failures**: Still dies early sometimes, but overall competent

The difference from CNN is striking - MLP looks like it understands the game, while CNN looked confused.

### Evaluation Commands

```bash
# Terminal 1: Game with rendering
VK_LOADER_DEBUG=error VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/radeon_icd.x86_64.json cargo run --release -- --fps 60

# Terminal 2: Evaluation
uv run --directory python python eval_sac.py results/sac_mlp_grpc/20251229_014052/models/final_model.zip --observation-mode with_thrower --episodes 5
```

---

## Infrastructure Validation

### gRPC + Parallel Servers: Working

The setup successfully achieved high throughput:

1. **start_grpc_servers.sh** launches N headless instances with unique sockets
2. **train_sac.py** connects to parallel envs via SubprocVecEnv
3. **Unix domain sockets** provide low-latency IPC (~0.1ms per step)
4. **850 FPS** during data collection phase (4 envs × ~200 FPS each)

### Key Code Changes

- `train_sac.py`: Added logic to connect to `_0.sock` for temp_env/eval_env when n_envs > 1
- `sac_mlp_grpc.yaml`: Config with `transport: grpc` and `n_envs: 4`
- `start_grpc_servers.sh`: Script to launch parallel headless instances

### Minor Issues

- `SubprocVecEnv` doesn't have `.envs` attribute, causing harmless warning when disabling training mode
- Results saved to `python/results/` not `results/` (path confusion)

---

## Takeaways

1. **gRPC + parallel servers work**: The infrastructure is validated
2. **MLP dramatically outperforms CNN**: 5x reward, 20x faster training
3. **State-based is the right choice**: When ground-truth state is available, use it
4. **~250 FPS sustained during training**: With parallel envs + gRPC + MLP (850 FPS during initial data collection)

---

## Files

- Config: `python/configs/sac_mlp_grpc.yaml`
- Results: `python/results/sac_mlp_grpc/20251229_014052/`
- Server script: `start_grpc_servers.sh`
- Training script: `python/train_sac.py`
