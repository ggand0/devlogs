# Devlog 022: Headless PPO Training Results

## Date: 2025-12-20

## Objective

Test the new headless mode with a 1M step PPO training run.

## Training Configuration

- **Algorithm**: PPO
- **Timesteps**: 1,000,000
- **Action Space**: basic_3d (3D continuous: move_x, move_y, sprint)
- **Level**: 2 (Hard) - 360° spawn angle, 0.5s spawn interval
- **Sprint**: 1.0 (2x speed at full sprint)
- **Spawn Angle**: ±30°

## Headless Mode Performance

| Metric | Windowed | Headless | Speedup |
|--------|----------|----------|---------|
| FPS | ~23 | ~45 | 2x |
| Training Time | ~12h (est) | 7h | 1.7x |

The speedup was limited by HTTP round-trip latency between Python and Bevy, not Bevy's tick rate. The `--fps 500` setting allows Bevy to tick faster, but actual training FPS was bottlenecked by the synchronous step() calls.

## Training Results

```
Final reward:    57.27
Peak reward:     71.41
Final ep length: 156
Peak ep length:  170
Best eval reward: 129.26
```

## Evaluation Results (10 episodes, deterministic)

```
Mean reward:     8.11 ± 62.28
Reward range:    [-71.43, 132.85]
Mean ep length:  106.9 ± 60.6 steps
Success rate:    0% (all episodes ended by collision)
```

### Episode Breakdown

| Episode | Reward | Steps | Notes |
|---------|--------|-------|-------|
| 1 | -29.45 | 70 | Early death |
| 2 | -40.49 | 59 | Early death |
| 3 | -71.43 | 29 | Very early death |
| 4 | -39.25 | 61 | Early death |
| 5 | 63.62 | 163 | Good survival |
| 6 | 41.20 | 139 | Moderate survival |
| 7 | 70.62 | 169 | Good survival |
| 8 | 132.85 | 226 | Best episode |
| 9 | -2.91 | 97 | Short survival |
| 10 | -43.62 | 56 | Early death |

## Analysis

**High variance**: The agent shows inconsistent behavior - sometimes surviving 200+ steps, other times dying within 30 steps. This suggests:

1. The policy hasn't converged to a robust strategy
2. The spawn angle randomness (±30°) creates situations the agent can't handle
3. 1M steps may not be enough for this difficulty level

**Comparison to SAC**: Previous SAC training at 2M steps achieved 586.12 best eval reward, significantly better than this PPO run. SAC's off-policy learning and sample efficiency appear to work better for this environment.

## Observations from Visual Evaluation

Watching the agent in windowed mode showed:
- The agent does move to dodge incoming projectiles
- Movement appears reasonable but not optimal
- Sometimes gets trapped in corners
- Doesn't seem to use sprint effectively in all situations

## Next Steps

1. Train for more steps (2M+) to see if performance improves
2. Try SAC in headless mode for comparison
3. Consider curriculum learning - start with narrower spawn angles
4. Tune reward function to encourage more proactive dodging

## Files

- Model: `results/ppo_level2_basic3d_2m/20251220_142642/models/best/best_model.zip`
- Config: `results/ppo_level2_basic3d_2m/20251220_142642/config.yaml`
- Plots: `results/ppo_level2_basic3d_2m/20251220_142642/plots/learning_curves.png`
