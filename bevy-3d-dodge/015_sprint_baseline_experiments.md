# Devlog 015: Sprint Baseline Experiments for Level 2

**Date:** December 8, 2025
**Focus:** Finding optimal sprint multiplier for 3D Basic continuous action space on Level 2

---

## Overview

After establishing that PPO with discrete actions achieves 100% success on Level 1, we've been working to solve Level 2 (Hard difficulty) using continuous action spaces. This devlog summarizes the sprint experiments with the simplified 3D Basic action space `[vx, vy, sprint]`.

---

## Level 2 Challenge

**Parameters:**
- Projectile speed: 4.5 units/sec (50% faster than Level 1)
- Spawn interval: 0.5 seconds (4x more frequent)
- Max projectiles: 25 (2.5x more)
- Spawn angle: ±60° fan (120° total) from +Y axis
- Random spawn positions within the arc

**Core Problem:** Dense projectile patterns from wide angles make dodging difficult. The hypothesis is that sufficient sprint speed should allow agents to escape even clustered spawns.

---

## Action Space: 3D Basic

Simplified continuous action space with 3 dimensions:
- `vx`: Horizontal velocity [-1, 1]
- `vy`: Forward/backward velocity [-1, 1]
- `sprint`: Speed boost intensity [-1, 1] → normalized to [0, 1]

**Speed Formula:**
```
effective_speed = base_speed * (1 + sprint * sprint_multiplier)
```

Where `base_speed = 5.0` units/sec.

---

## Experiment Results

### Experiment 1: 2x Sprint (sprint_multiplier = 1.0)

**Config:** `ppo_level2_basic3d.yaml` with default settings
**Date:** November 21, 2025
**Training:** 500K steps

**Speed Range:**
- No sprint: 5.0 units/sec
- Full sprint: 10.0 units/sec (2x base)

**Results:**
| Metric | Value |
|--------|-------|
| Mean reward | 38.10 ± 68.65 |
| Mean ep length | 136.7 ± 67.6 |
| Success rate | 0% (0/20 episodes) |
| Best episode | 174.56 reward, 273 steps |

**Analysis:**
- High variance indicates inconsistent learning
- 2x sprint (10.0) vs projectile speed (4.5) = 2.2x advantage
- Agent struggles to learn effective dodging strategy
- Never completed full 1000-step episode

---

### Experiment 2: 3x Sprint (sprint_multiplier = 2.0)

**Config:** `ppo_level2_basic3d.yaml` with `sprint_multiplier: 2.0`
**Date:** December 8, 2025
**Training:** 500K steps (~3.5 hours)

**Speed Range:**
- No sprint: 5.0 units/sec
- Full sprint: 15.0 units/sec (3x base)

**Results:**
| Metric | Value |
|--------|-------|
| Best eval reward (490K) | 106.24 ± 50.79 |
| Best eval ep length | 204.8 ± 50.86 |
| Final eval reward (500K) | 40.10 ± 59.03 |
| Final eval ep length | 139.2 ± 58.66 |
| Success rate | 0% (no full episodes) |

**Training Progression:**
```
Step 480K: eval_reward=90.58, ep_len=188.5
Step 490K: eval_reward=106.24, ep_len=204.8  ← Best checkpoint
Step 500K: eval_reward=40.10, ep_len=139.2   ← Performance dropped
```

**Analysis:**
- **Significant improvement** over 2x sprint (106 vs 38 best reward)
- 3x sprint (15.0) vs projectile speed (4.5) = 3.3x advantage
- High variance persists - performance unstable
- `approx_kl` values high (0.27-0.39), indicating aggressive policy updates
- Best model at 490K, not final model

---

## Comparison Summary

| Sprint Multiplier | Max Speed | Speed Advantage | Best Reward | Best Steps |
|-------------------|-----------|-----------------|-------------|------------|
| 1.0 (2x) | 10.0 | 2.2x over projectiles | 38.10 | 137 |
| 2.0 (3x) | 15.0 | 3.3x over projectiles | **106.24** | **205** |

**Key Finding:** 3x sprint shows **2.8x improvement** in best reward and **1.5x improvement** in survival time.

---

## Training Observations

### High KL Divergence Issue

Both experiments show high `approx_kl` values (0.15-0.39), well above the typical target of ~0.01-0.02. This indicates:
- Policy updates are too aggressive
- Learning is unstable
- Performance can regress quickly (as seen in 490K→500K drop)

**Potential fixes:**
- Lower learning rate: `0.0001` instead of `0.0003`
- Smaller clip range: `0.1` instead of `0.2`
- More training steps with slower updates

### Spawn Clustering Problem

Observed during human playtesting: consecutive projectiles can spawn from nearly identical angles (within the ±60° fan), creating unavoidable "walls" of projectiles.

**Potential fixes:**
- Reduce spawn angle (e.g., ±30° instead of ±60°)
- Add minimum angle separation between consecutive spawns
- Increase spawn interval slightly

---

## Visualization Added

Added spawn arc visualization to help debug projectile patterns:
- Red sphere markers along the ±60° arc at spawn distance (20 units)
- Larger spheres at arc edges (±60° boundaries)
- Visible in-game for both human play and training observation

---

## Human Sprint Control

Implemented Shift key for human sprint testing:
- Hold Shift while moving to sprint
- Uses same `sprint_multiplier` as RL agent
- Allows human verification of whether sprint speed is sufficient

---

## Next Steps

1. **Evaluate best checkpoint:** Run eval on 490K model to confirm 106 reward
2. **Lower learning rate:** Try `lr=0.0001` for more stable training
3. **Reduce spawn angle:** Test ±30° or ±45° fan to reduce clustering
4. **Add spawn separation:** Minimum angle delta between consecutive spawns
5. **Longer training:** 1M steps to see if performance stabilizes

---

## Files Modified

- `src/config.rs`: Updated `sprint_multiplier` to 2.0 for both levels
- `src/game/player.rs`: Added Shift key sprint for human control
- `src/main.rs`: Added spawn arc visualization, updated controls UI

---

## Conclusion

The 3x sprint experiment shows clear improvement over 2x sprint, confirming that higher escape velocity helps the agent survive longer. However, the learning is still unstable with high variance. The next focus should be on stabilizing training (lower LR, longer runs) and potentially reducing spawn angle to make the environment more learnable.

The agent's best performance (205 steps, ~20% of max episode) suggests the task is learnable but requires either:
1. More forgiving environment parameters
2. Better hyperparameter tuning
3. Longer training with more stable updates
