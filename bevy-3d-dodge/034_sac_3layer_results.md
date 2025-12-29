# SAC 3-Layer Architecture Results

## Experiment Overview

**Objective**: Test if a deeper network [256, 256, 256] can learn hierarchical strategy (aggressive when safe, cautious when in danger).

**Date**: December 29, 2024

**Duration**: ~2h 18m (estimated, similar to baseline)

---

## Configuration

```yaml
algorithm: SAC
transport: grpc
n_envs: 4
level: 2                          # Hard difficulty (±60° = 120° fan)
observation_mode: with_thrower    # 69-dim vector
action_space_type: basic_3d       # [vx, vy, sprint]

total_timesteps: 2000000          # 2M steps (same as baseline)
learning_rate: 0.0003
buffer_size: 100000
batch_size: 256
net_arch: [256, 256, 256]         # 3 layers (depth experiment)
```

---

## Results

### Training Metrics

| Metric | Value |
|--------|-------|
| Total timesteps | 2,000,000 |
| Best eval reward | 860.22 |
| Final eval reward | 690.01 ± 299.09 |

---

## Visual Evaluation (Best Model)

```
Episodes: 10, Mode: deterministic

Episode  1: ✗ Reward:   70.19, Steps:  170
Episode  2: ✗ Reward:  -24.21, Steps:   75
Episode  3: ✗ Reward:  274.57, Steps:  371
Episode  4: ✗ Reward:  300.03, Steps:  395
Episode  5: ✗ Reward:   -4.96, Steps:   92
Episode  6: ✗ Reward:    9.23, Steps:  109
Episode  7: ✗ Reward:  473.18, Steps:  568
Episode  8: ✗ Reward:  167.32, Steps:  263
Episode  9: ✗ Reward:   82.56, Steps:  182
Episode 10: ✗ Reward:  315.94, Steps:  412

Mean reward:       166.39 ± 158.97
Mean ep length:    263.7 ± 157.4
Success rate:      0% (0/10)
```

---

## Comparison to 2-Layer Baseline

| Metric | 2-Layer [256,256] | 3-Layer [256,256,256] | Difference |
|--------|-------------------|----------------------|------------|
| Best eval reward | 1007.27 | 860.22 | -15% |
| Final eval reward | 870.60 | 690.01 | -21% |
| Eval success rate | 40% | 0% | -40pp |
| Mean ep length | 720.0 | 263.7 | -63% |

**Result**: 3-layer architecture performed significantly worse than 2-layer baseline.

---

## Analysis

1. **Deeper network underperformed** - 3-layer achieved 0% success rate vs 40% for 2-layer
2. **Higher capacity requires more training** - The extra layer adds parameters that need more samples to converge
3. **2M steps insufficient for 3-layer** - The network likely needs 3-4M steps to match 2-layer performance
4. **Premature evaluation** - Best model at 860 eval reward is still in early learning phase

---

## Hypothesis

The 3-layer network has more representational capacity but:
- More parameters = slower convergence
- Same 2M budget splits gradients across more layers
- Need proportionally more training time

**Estimated requirement**: 3-4M steps for fair comparison with 2-layer 2M baseline.

---

## Next Steps

Options:
1. **Train 3-layer longer** - 4M steps to see if it eventually catches up
2. **Abandon depth, try width** - Test [512, 512] which has similar parameter count but proven architecture
3. **Stick with 2-layer** - Focus on other improvements (reward tuning, more training)

**Recommendation**: Try [512, 512] next - wider networks often train faster than deeper ones for RL tasks.

---

## Files

- Config: `python/configs/sac_3layer.yaml`
- Results: `results/sac_3layer/20251229_170537/`
- Best model: `results/sac_3layer/20251229_170537/models/best/best_model.zip`
