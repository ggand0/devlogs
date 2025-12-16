# Devlog 018: ±30° Stable Training - Extended to 2M Steps

## Date: 2025-12-11 (updated 2025-12-13)

## Objective

Train the ±30° spawn angle configuration with stability tweaks, extending from 1M to 2M steps to find the performance ceiling for PPO on this task.

## Configuration

Based on learnings from the ±20° experiment which showed training instability (high KL divergence ~0.06, clip fraction ~0.30), we applied stability tweaks:

```yaml
spawn_angle_degrees: 30      # ±30° = 60° total fan (best from previous experiments)
sprint_multiplier: 1.0       # 2x speed
total_timesteps: 1000000     # Extended to 1M steps

# Stability tweaks
learning_rate: 0.0001        # 3x lower (was 0.0003)
clip_range: 0.15             # Tighter clipping (was 0.2)
```

Config file: `python/configs/ppo_level2_basic3d_narrow_long.yaml`

## Results

### Training Metrics Comparison (1M vs 2M)

| Metric | 1M Steps | 2M Steps | Assessment |
|--------|----------|----------|------------|
| approx_kl | 0.008-0.017 | 0.016-0.032 | Increasing - signs of instability |
| clip_fraction | 0.10-0.22 | 0.20-0.29 | Higher clipping needed |
| explained_variance | 0.78-0.93 | 0.88-0.96 | Still strong |
| entropy | -4.97 | -5.42 | More stochastic policy |
| std | 1.28 | 1.49 | Wider action distribution |

### Performance Summary

| Metric | 1M Steps | 2M Steps |
|--------|----------|----------|
| Best Eval Reward | 170.61 | **177.03** |
| Final Eval Reward | 127.47 ± 76.92 | 100.45 ± 110.24 |
| Peak Training Reward | 144.06 | 152.26 |
| Final Training Reward | 124.25 | 151.61 |
| Peak Episode Length | 242 | 250 |
| Training Time | ~7 hours | ~14 hours total |

### Comparison with All Experiments

| Experiment | Sprint | Angle | Steps | Best Eval Reward |
|------------|--------|-------|-------|------------------|
| Exp 1 | 2x | ±60° | 500K | 38.10 |
| Exp 2 | 3x | ±60° | 500K | 106.24 |
| Exp 3 | 2x | ±30° | 500K | 130.59 |
| Exp 4 | 2x | ±20° | 1M | 106.90 |
| Exp 5a | 2x | ±30° | 1M | 170.61 |
| **Exp 5b** | **2x** | **±30°** | **2M** | **177.03** |

**Best eval improved** from 170.61 → 177.03 (+4%), but with diminishing returns and increasing instability.

## Analysis

### Why Stability Tweaks Helped (1M)

1. **Lower learning rate (0.0001)**: Slower but more consistent policy updates. The agent doesn't overshoot good solutions.

2. **Tighter clip range (0.15)**: Constrains how much the policy can change per update. Prevents the wild swings seen in the ±20° experiment.

3. **Result**: KL divergence stayed in healthy 0.008-0.017 range throughout the first 1M steps.

### Diminishing Returns at 2M Steps

The second 1M steps showed signs of reaching PPO's limits on this task:

1. **KL divergence doubled**: 0.016-0.032 range indicates the policy is changing more aggressively
2. **Clip fraction increased**: ~25% of updates being clipped vs ~15% before
3. **Higher variance**: Final eval 100.45 ± 110.24 shows inconsistent performance
4. **Only +4% improvement**: 170.61 → 177.03 for double the training time

The agent's policy is becoming more stochastic (higher entropy, wider std), which can indicate:
- Exploration is still finding new strategies
- Or the policy is destabilizing and becoming less deterministic

### High Variance in Evaluation

Both checkpoints show high eval variance, indicating:
- Some episodes the agent performs exceptionally well
- Other episodes it gets caught in difficult situations
- This is inherent to the dodging task where small mistakes compound

## Key Learnings

1. **Stability over speed**: Lower learning rate + tighter clipping allows longer training without destabilization
2. **±30° remains optimal**: Neither too predictable (±20°) nor too chaotic (±60°)
3. **Diminishing returns after 1M**: Going from 1M → 2M steps gave only +4% improvement
4. **PPO may be reaching its ceiling**: Increasing instability metrics suggest trying a different algorithm

## Files

- Config: `python/configs/ppo_level2_basic3d_narrow_long.yaml`
- Best model (2M): `results/ppo_level2_basic3d_narrow_long/20251211_024548/models/best/best_model.zip`
- Learning curves: `results/ppo_level2_basic3d_narrow_long/20251211_024548/plots/learning_curves.png`

## Next Steps

- **Try SAC (Soft Actor-Critic)**: Off-policy algorithm with automatic entropy tuning, may handle this continuous control task better
- Compare SAC @ 1M steps vs PPO @ 2M steps for sample efficiency
- Evaluate best PPO model visually to understand learned behaviors
