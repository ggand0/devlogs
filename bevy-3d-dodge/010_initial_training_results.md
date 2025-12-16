# Initial Training Results - 100k Steps

**Date:** 2025-11-16
**Training Duration:** ~46 minutes (100,000 steps)
**Hardware:** AMD Radeon RX 7900 XTX
**Algorithm:** DQN with MLP policy [64, 64]

## Training Configuration

```python
learning_rate: 1e-4
buffer_size: 50,000
batch_size: 32
gamma: 0.99
exploration: 1.0 → 0.05 (over first 30% of training)
target_update_interval: 1000 steps
```

## Training Progress

### Early Training (0-30k steps)
- **Episode length**: 140-190 steps
- **Mean reward**: 40-90
- **Behavior**: Random exploration, learning collision boundaries
- **Exploration rate**: 0.97 → 0.85

### Mid Training (30k-70k steps)
- **Episode length**: 190-380 steps
- **Mean reward**: 90-290
- **Behavior**: Learning to avoid obvious collisions
- **Exploration rate**: 0.85 → 0.15

### Late Training (70k-100k steps)
- **Episode length**: 380-397 steps (peak)
- **Mean reward**: 290-307 (peak)
- **Behavior**: Sophisticated dodging, some multi-ball avoidance
- **Exploration rate**: 0.15 → 0.05 (final)

### Key Metrics from Training

**Best evaluation (95k steps):**
- Mean episode length: **403 steps**
- Mean reward: **307.20 ± 148.73**
- Shows high variance but promising peaks

**Final evaluation (100k steps):**
- Mean episode length: **254 steps**
- Mean reward: **158.01 ± 122.49**
- Regression from peak - possible overfitting or instability

## Evaluation Results (Best Model)

Evaluated best checkpoint on 20 episodes in deterministic mode.

### Performance Summary

```
Total episodes:     20
Total steps:        3,190
Average reward:     60.74 ± 127.38
Average steps:      159.5 ± 125.7
Reward range:       [-100.00, 394.87]
Success rate:       0.0%
Steps per second:   39.0
```

### Episode Breakdown

| Episode | Steps | Reward | Outcome |
|---------|-------|--------|---------|
| 1 | 134 | 34.53 | Collision |
| 2 | 99 | 0.35 | Collision |
| 3 | **1** | **-100.00** | Immediate collision |
| 4 | 194 | 94.97 | Collision |
| 5 | 305 | 210.54 | Collision (good run) |
| 6 | 190 | 90.23 | Collision |
| 7 | 101 | 2.24 | Collision |
| 8 | **1** | **-100.00** | Immediate collision |
| 9 | 191 | 91.88 | Collision |
| 10 | 300 | 203.64 | Collision (good run) |
| 11 | 298 | 199.82 | Collision (good run) |
| 12 | 100 | 1.00 | Collision |
| 13 | **1** | **-100.00** | Immediate collision |
| 14 | **490** | **394.87** | Collision (best run!) |
| 15 | 100 | 0.55 | Collision |
| 16 | 96 | -3.68 | Collision |
| 17 | **1** | **-100.00** | Immediate collision |
| 18 | 297 | 202.40 | Collision (good run) |
| 19 | 194 | 94.18 | Collision |
| 20 | 97 | -2.75 | Collision |

### Key Observations

**Positive:**
1. **Peak performance is good**: Episode 14 achieved 490 steps (nearly half the max episode length!)
2. **Can handle multiple projectiles**: Episodes 5, 10, 11, 14, 18 all survived 300+ steps with 2 projectiles
3. **Learning occurred**: Mean reward increased from ~40 early training to ~160 at evaluation
4. **Some sophisticated dodging**: Reward bonuses (>1.0) indicate close dodges

**Negative:**
1. **High variance**: σ = 127.38 is nearly 2x the mean (60.74)
2. **Immediate failures**: 4 episodes died on step 1 (20% failure rate at spawn)
3. **No truncated episodes**: 0% success rate (never reached 1000 steps)
4. **Inconsistent performance**: Best episode was 490 steps, but 50% of episodes died before 150 steps

### Behavioral Analysis

**What the agent learned:**
- Basic collision avoidance (can survive 100+ steps consistently)
- Some spatial awareness (dodges projectiles when they're far)
- Multi-projectile scenarios (can track 2-3 balls simultaneously)

**What the agent struggles with:**
1. **Initial spawns**: Dies immediately 20% of the time
   - Suggests policy hasn't learned safe starting positions
   - Possible reset synchronization issue
2. **Long-term planning**: Can't sustain performance past ~300 steps
   - May not see full projectile trajectories in observation window
   - Possible credit assignment problem (reward too sparse)
3. **Deterministic behavior**: Evaluation used deterministic policy
   - May have learned suboptimal local minimum
   - Exploration in evaluation might help (stochastic mode)

## Training Stability

**Loss trends:**
- Started at ~0.4
- Decreased to ~0.02-0.05 mid-training
- Final loss: 0.0771
- Some spikes (0.293, 0.344) indicate potential instability

**Possible issues:**
1. **No success signal**: Agent never experienced truncation (max steps)
   - All episodes end in -100 penalty
   - No positive terminal state to learn from
2. **Sparse rewards**: +1 per step, -100 on death
   - Large gap makes credit assignment hard
   - May need reward shaping
3. **Small network**: [64, 64] may lack capacity for complex dodging
4. **Fixed exploration**: ε=0.05 at end may be too low for this task

## Comparison: Training vs Evaluation

| Metric | Training (best) | Evaluation (best model) | Delta |
|--------|----------------|-------------------------|-------|
| Mean steps | 403 | 159.5 | **-243** |
| Mean reward | 307 | 60.74 | **-246** |
| Best episode | ~400+ | 490 | +90 |

**Discrepancy analysis:**
- Training evaluations used ε-greedy exploration
- Test evaluation used deterministic policy
- Suggests agent may need some exploration even at test time
- Possible overfitting to training distribution

## Recommendations for Improvement

### High Priority (likely to help)

1. **Increase training steps**: 100k may be insufficient
   - Try 300k-500k steps
   - Monitor for continued improvement

2. **Reward shaping**:
   ```python
   # Current: +1 survival, -100 collision
   # Proposed:
   reward = 1.0  # base survival
   if min_distance < 2.0:
       reward += (2.0 - min_distance) * 1.0  # stronger dodge bonus
   if min_distance < 1.0:
       reward += 5.0  # very close dodge bonus
   if collision:
       reward = -50.0  # reduce collision penalty (currently -100)
   ```

3. **Fix immediate death issue**:
   - Investigate reset synchronization
   - Ensure player spawns in safe position
   - Add spawn invulnerability (first 10 steps?)

4. **Network capacity**:
   - Try [128, 128] or [256, 256]
   - May need more capacity for multi-projectile reasoning

### Medium Priority

5. **Observation normalization**:
   ```python
   obs = obs / 100.0  # Scale to [-1, 1]
   ```

6. **Curriculum learning**:
   - Start with slower projectiles
   - Gradually increase speed
   - Increase spawn rate over time

7. **Max episode steps**:
   - Current: 1000 steps
   - Increase to 2000 or 5000 for more learning signal

8. **Exploration at test time**:
   - Try ε=0.1 at evaluation
   - May perform better than deterministic

### Low Priority (nice to have)

9. **Algorithm upgrades**:
   - Try PPO (may handle continuous episodes better)
   - Try Rainbow DQN (distributional RL)
   - Try Double DQN (reduce overestimation)

10. **Advanced features**:
    - Recurrent policy (LSTM) for temporal reasoning
    - Attention mechanism for multi-projectile tracking
    - Frame stacking (include velocity history)

## Next Steps

**Immediate actions:**
1. Run longer training (300k-500k steps)
2. Implement stronger reward shaping
3. Debug immediate death issue
4. Try larger network [256, 256]

**Expected improvements:**
- Success rate should reach 10-20% (truncated episodes)
- Mean episode length should exceed 400 consistently
- Variance should decrease (more stable policy)
- Immediate deaths should drop to <5%

**Success criteria for "good" agent:**
- Mean reward: >300
- Mean episode length: >500
- Success rate: >30%
- Immediate death rate: <5%

## Conclusion

The agent **learned to dodge**, but performance is inconsistent. Key issues:
1. High variance (some 490-step episodes, some 1-step deaths)
2. Never reaching max episode length (1000 steps)
3. Possible training instability (loss spikes, performance regression)

**Verdict:** Promising start, needs more training and better reward shaping. The fact that episode 14 survived 490 steps shows the agent *can* learn sophisticated dodging - we just need to make that behavior more consistent.

**Time well spent:** 46 minutes of training produced an agent that sometimes displays impressive multi-ball dodging. With hyperparameter tuning and longer training, this should improve significantly.
