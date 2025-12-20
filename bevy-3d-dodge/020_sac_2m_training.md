# Devlog 020: SAC 2M Training - Continued Improvement

## Date: 2025-12-18

## Objective

Continue SAC training from 1M to 2M steps to explore the performance ceiling.

## Configuration

Same as 1M training (devlog 019):

```yaml
algorithm: SAC
spawn_angle_degrees: 30
sprint_multiplier: 1.0
total_timesteps: 2000000  # Extended from 1M
```

## Results

### Training Progression

| Checkpoint | Best Eval Reward | Change |
|------------|------------------|--------|
| 550K | 357.72 | - |
| 1M | 422.54 | +18% |
| **2M** | **586.12** | **+39%** |

### 2M Final Stats

```
Best Eval Reward:   586.12 +/- 265.60
Best Eval Length:   662.40 +/- 234.60
Final Eval Reward:  441.00 +/- 305.68
Final Eval Length:  518.40 +/- 267.94
Training Time:      ~12 hours (for 1M→2M segment)
Total Time:         ~24 hours cumulative
```

### Algorithm Comparison (Updated)

| Algorithm | Steps | Best Eval Reward | vs PPO |
|-----------|-------|------------------|--------|
| PPO | 2M | 177.03 | baseline |
| SAC | 1M | 422.54 | 2.4x better |
| **SAC** | **2M** | **586.12** | **3.3x better** |

### Training Metrics at 2M

- `actor_loss`: ~22-25 (stable)
- `critic_loss`: ~8-22 (normal variance)
- `ent_coef`: 0.028 (settled, good exploration-exploitation balance)
- `fps`: 23 (consistent throughput)
- `rollout ep_rew_mean`: ~255 (training episodes)

## Analysis

### Continued Learning

SAC showed no signs of plateauing:
- **39% improvement** from 1M to 2M steps
- Best eval jumped from 422 → 586
- Episode lengths increased from ~500 → 662 steps average

### Stability

Unlike PPO which destabilized around 1.5-2M steps:
- Entropy coefficient remained stable at ~0.028
- No explosion in actor/critic losses
- Consistent FPS throughout training

### Why SAC Keeps Improving

1. **Replay buffer**: 1M transitions means diverse training data
2. **Off-policy**: Can revisit rare situations repeatedly
3. **Soft updates**: tau=0.005 prevents catastrophic forgetting
4. **Auto entropy**: Maintains appropriate exploration level

## Key Findings

1. **SAC scales well**: 2M steps still showing improvement (39% gain)
2. **3.3x better than PPO**: At same step count, SAC dominates
3. **No instability**: Unlike PPO, SAC remains stable at high step counts
4. **Room for more**: Learning curves suggest 3M+ could yield further gains

## Files

- Best model (2M): `results/sac_level2_basic3d/20251213_024723/models/best/best_model.zip`
- Final model: `results/sac_level2_basic3d/20251213_024723/models/final_model.zip`
- Learning curves: `results/sac_level2_basic3d/20251213_024723/plots/learning_curves.png`

## Next Steps

- Visual evaluation of 2M model to assess learned behavior
- Consider 3M training if curves still trend upward
- Explore curriculum learning (Level 1 → Level 2)
- Experiment with reward shaping for more consistent performance

## Conclusion

SAC at 2M steps achieves **586.12 best eval reward**, a 3.3x improvement over PPO at the same step count. The algorithm shows no signs of plateauing, suggesting even longer training could push performance higher. SAC is clearly the superior choice for this continuous control dodging task.
