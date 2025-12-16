# Devlog 017: Very Narrow Spawn Angle Experiment (±20°)

**Date:** December 9, 2025
**Focus:** Testing ±20° spawn angle to see if narrower is better

---

## Overview

Following the successful ±30° experiment (best eval reward: 130.59), we tested an even narrower spawn angle of ±20° to see if reducing spawn variability further would help.

**Hypothesis:** If ±30° outperformed ±60°, then ±20° might be even better by further reducing unfair clustering.

---

## Configuration

| Parameter | Value |
|-----------|-------|
| Spawn angle | ±20° (40° total fan) |
| Sprint multiplier | 1.0 (2x speed) |
| Total steps | 500K |
| Learning rate | 0.0003 |
| Action space | basic_3d [vx, vy, sprint] |

Config file: `ppo_level2_basic3d_narrow20.yaml`

---

## Results

### Training Metrics (500K steps)

| Metric | Value |
|--------|-------|
| Best eval reward | 77.61 |
| Final eval reward | 56.25 |
| Best eval ep length | ~163 |
| Final ep length | 155 |
| Peak rollout reward | 58.90 |

### Comparison with Previous Experiments

| Experiment | Sprint | Angle | Best Eval Reward | Best Steps |
|------------|--------|-------|------------------|------------|
| Exp 1 | 2x | ±60° | 38.10 | 137 |
| Exp 2 | 3x | ±60° | 106.24 | 205 |
| Exp 3 | 2x | ±30° | **130.59** | **227** |
| **Exp 4** | **2x** | **±20°** | **77.61** | **163** |

**Result: ±20° performed ~40% worse than ±30°**

---

## Observations

### Learning Curves
The learning curves suggest the agent may not have fully converged - reward was still trending upward at 500K steps. Additional training might improve results.

### Agent Behavior
During evaluation, the agent exhibited a **corner-camping strategy**:
1. Moves toward a corner of the play area
2. Makes small micro-adjustments to dodge nearby projectiles
3. Eventually gets trapped and unable to escape

This is a **local optimum** - the strategy works short-term but fails long-term because:
- All projectiles come from roughly the same direction (±20° is very narrow)
- Agent doesn't need to learn dynamic movement patterns
- Once cornered, there's no escape route

### Why ±20° is Worse Than ±30°

1. **Too predictable:** Narrow spawn fan means projectiles always come from similar angles
2. **Encourages passive strategies:** Agent can "camp" since threats are predictable
3. **Less training diversity:** Agent doesn't experience varied scenarios
4. **No incentive for zig-zag movement:** The ±30° agent learned dynamic dodging; ±20° agent just slides to corner

---

## Training Stability

KL divergence remained high throughout training:
- `approx_kl`: 0.05 - 0.10 (target ~0.01-0.02)
- `clip_fraction`: 0.30 - 0.38 (high clipping)

This indicates unstable policy updates, though this was also present in previous experiments.

---

## Key Insight

**There's a sweet spot for spawn angle variability:**
- ±60° (default): Too much variability, creates unavoidable clustering
- ±30°: Sweet spot - enough variability to learn robust dodging
- ±20°: Too narrow - encourages passive corner-camping

The ±30° configuration forces the agent to learn **active dodging** because threats come from varied angles. The ±20° configuration allows the agent to be **lazy** and just retreat to a corner.

---

## Next Steps

1. **Extended ±30° training:** Run 1M steps with the best configuration to see if it converges further
2. **Resume ±20° training:** Try 500K more steps to see if it breaks out of the local optimum
3. **Test ±45°:** Middle ground between 30° and 60° might be worth exploring

---

## Files

- Config: `python/configs/ppo_level2_basic3d_narrow20.yaml`
- Results: `results/ppo_level2_basic3d_narrow20/20251209_004127/`
- Learning curves: `results/ppo_level2_basic3d_narrow20/20251209_004127/plots/learning_curves.png`

---

## Conclusion

The ±20° experiment demonstrates that **narrower isn't always better**. The spawn angle needs enough variability to force the agent to learn robust, dynamic dodging behavior. The ±30° configuration remains the best performer, achieving both the highest reward and the most desirable agent behavior (zig-zag dodging vs corner-camping).
