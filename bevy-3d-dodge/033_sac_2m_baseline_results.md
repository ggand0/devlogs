# SAC 2M Baseline Results

## Experiment Overview

**Objective**: Test if the current [256, 256] architecture just needs more training time to master Level 2.

**Date**: December 29, 2024

**Duration**: 2h 18m 54s

---

## Configuration

```yaml
algorithm: SAC
transport: grpc
n_envs: 4
level: 2                          # Hard difficulty (±60° = 120° fan)
observation_mode: with_thrower    # 69-dim vector
action_space_type: basic_3d       # [vx, vy, sprint]

total_timesteps: 2000000          # 2M steps (2x previous)
learning_rate: 0.0003
buffer_size: 100000
batch_size: 256
net_arch: [256, 256]
```

---

## Results

### Final Metrics

| Metric | Value |
|--------|-------|
| Total timesteps | 2,000,000 |
| Training time | 2h 18m 54s |
| Best eval reward | 1007.27 |
| Final eval reward | 870.60 ± 204.93 |
| Final eval ep length | 900.80 ± 165.37 |

### Training Metrics (rollout)

| Metric | Value |
|--------|-------|
| Final reward | 184.51 |
| Peak reward | 303.15 |
| Final ep length | 279 |
| Peak ep length | 397 |

### Model Stats at End

| Metric | Value |
|--------|-------|
| actor_loss | -11.3 |
| critic_loss | 27.7 |
| ent_coef | 0.0261 |
| n_updates | 497,499 |

---

## Comparison to 1M Baseline

| Metric | 1M steps | 2M steps | Change |
|--------|----------|----------|--------|
| Training time | 1h 9m | 2h 18m | +2x |
| Best eval reward | 874.66 | 1007.27 | +15% |
| Final eval reward | 660.48 | 870.60 | +32% |
| Final eval ep length | 715 | 901 | +26% |

**Key improvement**: Best eval hit 1007, meaning the agent achieved a perfect episode (max 1000 steps + small reward variance).

---

## Analysis

1. **2M steps shows significant improvement** over 1M steps
2. **Best eval 1007** indicates the agent CAN solve Level 2 perfectly
3. **High variance** (±204) suggests inconsistent performance - sometimes perfect, sometimes dies early
4. **Rollout metrics lower than eval** - training episodes are shorter, likely due to exploration

---

## Visual Evaluation (Final Model)

```
Episodes: 10, Mode: deterministic

Episode  1: ✗ Reward:  230.24, Steps:  328
Episode  2: ✗ Reward:  369.60, Steps:  463
Episode  3: ✗ Reward:  710.29, Steps:  805
Episode  4: ✗ Reward:  495.06, Steps:  581
Episode  5: ✓ Reward: 1001.30, Steps: 1000
Episode  6: ✗ Reward:  693.16, Steps:  782
Episode  7: ✓ Reward: 1003.41, Steps: 1000
Episode  8: ✓ Reward: 1010.39, Steps: 1000
Episode  9: ✓ Reward: 1003.53, Steps: 1000
Episode 10: ✗ Reward:  142.83, Steps:  241

Mean reward:       665.98 ± 322.83
Mean ep length:    720.0 ± 281.6
Success rate:      40% (4/10)
```

**Observations**:
- Agent shows competent dodging behavior
- Can achieve perfect episodes (4/10 reached 1000 steps)
- High variance - sometimes dies early (ep 10: 241 steps)
- Failures seem to be early mistakes rather than late-game collapse

---

## Next Steps

Options to reduce variance and improve consistency:
1. **Wider architecture** - Try [512, 512] (sac_wide_arch.yaml ready)
2. **More training** - 3M or 4M steps
3. **Reward tuning** - Increase dodge bonus to encourage riskier play

---

## Files

- Config: `python/configs/sac_2m_baseline.yaml`
- Results: `results/sac_2m_baseline/20251229_142239/`
- Best model: `results/sac_2m_baseline/20251229_142239/models/best/best_model.zip`
- Final model: `results/sac_2m_baseline/20251229_142239/models/final_model.zip`
