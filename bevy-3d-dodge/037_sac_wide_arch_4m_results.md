# SAC Wide Architecture [512, 512] 4M Results

## Experiment Overview

**Objective**: Test if wider networks improve performance over baseline [256, 256].

**Date**: December 30, 2024

**Training Time**: 4h 36m

---

## Configuration

```yaml
algorithm: SAC
transport: grpc
n_envs: 4
level: 2                          # Hard difficulty (±60° = 120° fan)
observation_mode: with_thrower    # 69-dim vector
action_space_type: basic_3d       # [vx, vy, sprint]

total_timesteps: 4000000          # 4M steps
learning_rate: 0.0003
buffer_size: 100000
batch_size: 256
net_arch: [512, 512]              # 2x wider hidden layers (~4x params)
```

---

## Results

### Training Metrics

| Metric | Value |
|--------|-------|
| Best eval reward | 932.30 |
| Final eval reward | 623.78 |
| Final rollout reward | 209.81 |
| Final ep length | 306 |
| Training time | 4h 36m |

### Learning Curve Analysis

The evaluation reward showed **extreme instability**:
- Peaked at 932 around 0.5-1M steps
- Oscillated wildly between 400-900 throughout training
- Never converged to a stable policy
- Final performance (624) significantly below peak

This instability suggests the wider network is harder to optimize and prone to forgetting.

---

## Architecture Comparison (All Experiments)

| Architecture | Params (est.) | Steps | Best Eval | Final Eval | Stability |
|-------------|---------------|-------|-----------|------------|-----------|
| [256, 256] | ~132K | 2M | **1007** | 871 | Moderate |
| [256, 256, 256] | ~197K | 4M | 907 | 807 | Low |
| [512, 512] | ~526K | 4M | 932 | 624 | Very Low |

**Key findings**:
1. **Baseline [256, 256] is optimal** - Best peak performance AND most stable
2. **More parameters hurt** - Both deeper and wider networks underperform
3. **Wider = more unstable** - [512, 512] showed the highest variance
4. **2x training didn't help** - 4M steps for larger networks still worse than 2M baseline

---

## Why Wider Networks Fail

Similar to the depth experiment (see devlog 035), wider networks face:

1. **Overfitting to recent experiences**: More parameters = more capacity to memorize
2. **Optimization difficulty**: Larger networks have more complex loss landscapes
3. **Catastrophic forgetting**: Network forgets good behaviors when updating on new data
4. **Sample inefficiency**: More parameters require more data to generalize

The oscillating eval curve suggests the network keeps "forgetting" good policies.

---

## Conclusion

The architecture experiments are complete. Results consistently show:

> **Simple [256, 256] architecture is optimal for this continuous control task.**

Scaling up the network (depth or width) provides no benefit and introduces instability.

### Recommendations

1. **Keep [256, 256] baseline** - It's already the right size
2. **Focus elsewhere** - Architecture isn't the bottleneck
3. **Try reward tuning** - Modify dodge bonus to encourage better behavior
4. **Try curriculum learning** - Pre-train on easier levels

---

## Files

- Config: `python/configs/sac_wide_arch.yaml`
- Results: `python/results/sac_wide_arch/20251230_014206/`
- Best model: `python/results/sac_wide_arch/20251230_014206/models/best/best_model.zip`
- Learning curve: `python/results/sac_wide_arch/20251230_014206/plots/learning_curves.png`
