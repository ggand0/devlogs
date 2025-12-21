# Devlog 023: Best Models Reference

## Date: 2025-12-21

## Purpose

Quick reference for the best trained models and their evaluation commands.

## Best Models

### SAC 2M (Best Overall)

**Path:**
```
results/sac_level2_basic3d/20251213_024723/models/best/best_model.zip
```

**Training Config:**
- Algorithm: SAC
- Steps: 2,000,000
- Action Space: basic_3d (3D continuous)
- Level: 2
- Spawn Angle: ±30° (60° total)
- Sprint Multiplier: 1.0

**Performance:**
- Best Eval Reward: **586.12**
- Best Eval Length: 662.40 steps
- Success Rate: 15% (episodes reaching 1000 step limit)

**Evaluate:**
```bash
# Terminal 1: Start game
VK_LOADER_DEBUG=error VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/radeon_icd.x86_64.json cargo run --release

# Terminal 2: Evaluate
uv run python python/eval_sac.py results/sac_level2_basic3d/20251213_024723/models/best/best_model.zip --episodes 10 --spawn-angle 30
```

---

### PPO 2M

**Path:**
```
results/ppo_level2_basic3d_narrow_long/20251211_024548/models/best/best_model.zip
```

**Training Config:**
- Algorithm: PPO
- Steps: 2,000,000
- Action Space: basic_3d (3D continuous)
- Level: 2
- Spawn Angle: ±30° (60° total)
- Sprint Multiplier: 1.0

**Performance:**
- Best Eval Reward: **177.03**
- Shows instability at higher step counts

**Evaluate:**
```bash
# Terminal 1: Start game
VK_LOADER_DEBUG=error VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/radeon_icd.x86_64.json cargo run --release

# Terminal 2: Evaluate
uv run python python/eval_ppo.py results/ppo_level2_basic3d_narrow_long/20251211_024548/models/best/best_model.zip --episodes 10 --spawn-angle 30
```

---

## Comparison Summary

| Model | Steps | Best Eval Reward | Relative Performance |
|-------|-------|------------------|---------------------|
| SAC 2M | 2M | 586.12 | **3.3x better** |
| PPO 2M | 2M | 177.03 | baseline |

## Important Notes

1. **Spawn Angle Mismatch**: Both models were trained with `spawn_angle_degrees=30`. The default Level 2 config uses 60°, so always pass `--spawn-angle 30` when evaluating.

2. **Algorithm Recommendation**: SAC is significantly better for this continuous control task due to:
   - Off-policy learning with replay buffer
   - Automatic entropy tuning
   - Better sample efficiency (4.8x)

## Re-Evaluation Results (2025-12-21)

### SAC 2M (10 episodes)

```
Mean reward:           487.70 ± 314.10
Reward range:          [123.06, 1004.64]
Mean episode length:   564.4 ± 280.9 steps
Length range:          [222, 1000] steps
Success rate:          20.0% (2/10 episodes)

Episode Breakdown:
✓ Success (2): Episodes 4, 10
✗ Failed  (8): Episodes 1, 2, 3, 5, 6, 7, 8, 9
```

**Observations:** High variance persists. Agent achieves 1000-step cap in 20% of episodes but performance varies wildly based on projectile spawn patterns.

### PPO 2M (10 episodes)

```
Mean reward:           96.07 ± 91.67
Reward range:          [-37.03, 199.49]
Mean episode length:   194.9 ± 90.9 steps
Length range:          [63, 299] steps
Success rate:          0.0% (0/10 episodes)

Episode Breakdown:
✗ Failed  (10): All episodes
```

**Observations:** Consistently underperforms SAC. Never reaches 1000-step cap. Best episode only 299 steps.

### Comparison

| Metric | SAC 2M | PPO 2M |
|--------|--------|--------|
| Mean Reward | 487.70 | 96.07 |
| Success Rate | 20% | 0% |
| Best Episode | 1004.64 | 199.49 |
| Worst Episode | 123.06 | -37.03 |

SAC achieves **5x higher mean reward** and is the only algorithm to reach the 1000-step limit.

## Related Devlogs

- [018_narrow30_stable_1m.md](018_narrow30_stable_1m.md) - PPO 2M training details
- [019_sac_breakthrough.md](019_sac_breakthrough.md) - SAC 1M results
- [020_sac_2m_training.md](020_sac_2m_training.md) - SAC 2M training details
