# SAC 3-Layer 4M Results

## Experiment Overview

**Objective**: Test if the 3-layer [256,256,256] network can catch up to 2-layer baseline with more training time.

**Date**: December 29-30, 2024

**Total Training Time**: ~5h (2h18m for 2M + 2h47m for additional 2M)

---

## Configuration

```yaml
algorithm: SAC
transport: grpc
n_envs: 4
level: 2                          # Hard difficulty (±60° = 120° fan)
observation_mode: with_thrower    # 69-dim vector
action_space_type: basic_3d       # [vx, vy, sprint]

total_timesteps: 4000000          # 4M total (2M + 2M resumed)
learning_rate: 0.0003
buffer_size: 100000
batch_size: 256
net_arch: [256, 256, 256]         # 3 layers (depth experiment)
```

---

## Results

### Training Metrics (4M total)

| Metric | 2M Checkpoint | 4M Final |
|--------|---------------|----------|
| Best eval reward | 860.22 | 907.33 |
| Final eval reward | 690.01 | 807.05 |
| Training time | 2h 18m | +2h 47m |

### Visual Evaluation (Best Model @ 4M)

```
Episodes: 10, Mode: deterministic

Episode  1: ✗ Reward:  247.97, Steps:  343
Episode  2: ✗ Reward:   45.64, Steps:  145
Episode  3: ✗ Reward:    1.47, Steps:  101
Episode  4: ✗ Reward:  260.41, Steps:  357
Episode  5: ✗ Reward:  875.70, Steps:  966
Episode  6: ✗ Reward:  443.94, Steps:  539
Episode  7: ✗ Reward:  636.09, Steps:  726
Episode  8: ✗ Reward:  337.21, Steps:  435
Episode  9: ✗ Reward:  226.73, Steps:  326
Episode 10: ✗ Reward:  634.44, Steps:  721

Mean reward:       370.96 ± 262.95
Mean ep length:    465.9 ± 259.2
Success rate:      0% (0/10)
```

---

## Comparison: 3-Layer vs 2-Layer Baseline

| Metric | 2-Layer [256,256] @ 2M | 3-Layer [256,256,256] @ 4M |
|--------|------------------------|---------------------------|
| Best eval reward | 1007.27 | 907.33 |
| Final eval reward | 870.60 | 807.05 |
| Visual success rate | 40% (4/10) | 0% (0/10) |
| Visual mean reward | 665.98 | 370.96 |
| Training time | 2h 18m | 5h 5m |

**Result**: Even with 2x training time, 3-layer still underperforms 2-layer baseline.

---

## Behavioral Observations

The 3-layer model developed a distinct strategy:
- **Wall-hugging rotation**: Tends to rotate counter-clockwise or clockwise while staying near arena edges
- **Occasional interesting movements**: Sometimes shows dynamic dodging
- **Less consistent**: Higher variance in episode outcomes

This suggests the deeper network learned a qualitatively different policy, but one that's less effective for survival.

---

## Analysis

1. **More training helped but not enough**: Best eval improved from 860 to 907 (+5.5%)
2. **Still significantly behind baseline**: 907 vs 1007 (-10%)
3. **Visual performance gap is larger**: 0% vs 40% success rate
4. **Different strategy emerged**: Wall-rotation vs active center dodging

### Possible Explanations

1. **Primacy bias**: The extra layer may have "locked in" suboptimal early representations
2. **Local minimum**: The rotation strategy is a local optimum that's hard to escape
3. **Needs even more training**: 4M may still not be enough for 3-layer to fully converge

---

## Conclusion

The 3-layer experiment confirms the RL research findings:
- Deeper networks are harder to train
- They may converge to different (inferior) strategies
- Width is generally better than depth for continuous control

**Recommendation**: Abandon 3-layer, try [512, 512] wide architecture instead.

---

## Files

- Config: `python/configs/sac_3layer_4m.yaml`
- Results: `results/sac_3layer_4m/20251229_223605/`
- Best model: `results/sac_3layer_4m/20251229_223605/models/best/best_model.zip`
- Previous 2M checkpoint: `results/sac_3layer/20251229_170537/`
