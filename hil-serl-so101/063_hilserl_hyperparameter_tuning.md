# HIL-SERL Hyperparameter Tuning

## Problem

After 120 episodes of training with `action_scale=0.02`, the robot arm wouldn't leave the home position. The entropy coefficient α dropped to 0.03, meaning almost no exploration.

## Root Cause Analysis

The policy learned to output near-zero actions, and with low entropy (α=0.03), there was no exploration to discover better behaviors.

## Research: Original HIL-SERL Parameters

### Sources
- [HIL-SERL Paper](https://hil-serl.github.io/static/hil-serl-paper.pdf)
- [HIL-SERL GitHub (rail-berkeley)](https://github.com/rail-berkeley/hil-serl)
- [HuggingFace LeRobot HIL-SERL Guide](https://huggingface.co/docs/lerobot/hilserl)

### Parameter Comparison

| Parameter | Original HIL-SERL | HF LeRobot Example | Our Config (before) | Our Config (after) |
|-----------|-------------------|-------------------|---------------------|-------------------|
| **action_scale** | 0.025-0.03m | 0.02m | 0.02m | 0.02m |
| **target_entropy** | -dim/2 = **-2** | null (-dim = -4) | null (-4) | **-2.0** |
| temperature_init | 1.0 (pixel-based) | 0.01 | 0.2 | 0.2 |
| discount | 0.95 | 0.97 | 0.97 | 0.97 |
| use_backup_entropy | False | True | True | True |
| temperature_lr | 3e-4 | 3e-4 | 3e-4 | 3e-4 |
| critic_lr | 3e-4 | 3e-4 | 3e-4 | 3e-4 |
| actor_lr | 3e-4 | 3e-4 | 3e-4 | 3e-4 |
| batch_size | - | 256 | 256 | 256 |
| utd_ratio | - | 2 | 2 | 2 |
| critic_target_update_weight (tau) | 0.005 | 0.005 | 0.005 | 0.005 |
| num_critics | 2 | 2 | 2 | 2 |

### Key Findings

1. **target_entropy**: Original HIL-SERL uses `-dim/2` formula, not `-dim`. For 4D actions (xyz + gripper), this is -2, not -4. Higher target entropy = more exploration.

2. **temperature_init**: HF recommends 0.01, original uses 1.0 for pixel-based. HF doc notes: "setting it too high can make human interventions ineffective."

3. **discount**: Original uses 0.95 (shorter horizon focus), HF example uses 0.97.

## Why The Arm Wasn't Moving: Root Cause Analysis

Two factors combined to cause the problem:

### 1. Wrong target_entropy (-4 instead of -2)
- SAC automatically adjusts α (entropy coefficient) to achieve target entropy
- Lower target (-4) = policy allowed to be more deterministic = α drops to 0.03
- Higher target (-2) = policy forced to maintain randomness = α stays higher (~0.1-0.3)
- With α=0.03, policy outputs became concentrated near zero (no exploration)

### 2. Small action_scale (0.02m)
- Policy outputs are in [-1, 1] range
- With α=0.03, outputs cluster around 0 (say ±0.1)
- Actual movement = 0.1 × 0.02m = **2mm per step**
- 2mm is basically invisible/imperceptible

### Combined Effect
```
Wrong target_entropy → α drops to 0.03 → policy outputs ~0 →
tiny outputs × small scale = no visible movement
```

### Why Larger action_scale Was a Workaround, Not a Fix
- **action_scale = 0.1**: Amplifies tiny outputs (0.1 × 0.1m = 1cm) → visible movement, but policy still not exploring properly
- **target_entropy = -2**: Forces policy to output varied actions → proper exploration, works even with 0.02 scale

Increasing action_scale was a **bandaid** masking the real problem. The fundamental fix is correcting target_entropy so the policy explores properly.

## Solution Applied Mid-Training

Changed `target_entropy` from `null` (defaults to -4) to `-2.0`. This forces SAC to maintain higher entropy, which will:
1. Increase α (entropy coefficient) from 0.03 toward higher values
2. Add more randomness to policy outputs
3. Encourage exploration away from near-zero actions

## Additional Fixes in This Session

### 1. Image Writer Float Range Fix
Images were being saved as float [0-255] instead of uint8, causing checkpoint failures.

**File**: `lerobot/datasets/image_writer.py`

```python
# Before: only handled float [0-1]
# After: handles both float [0-1] and float [0-255]
if max_ > 1.0 + 1e-5:
    # Float image with [0-255] range - convert directly to uint8
    image_array = np.clip(image_array, 0.0, 255.0).astype(np.uint8)
else:
    # Float image with [0-1] range
    image_array = np.clip(image_array, 0.0, 1.0)
    image_array = (image_array * 255).astype(np.uint8)
```

### 2. Episode Count Logging
Added episode counter to actor and learner logs for better progress tracking.

**Actor log format**:
```
[ACTOR] Episode 1 done | step=150 | reward=0.85
```

**Learner log format**:
```
[LEARNER] Episode 1 | step=150 | reward=0.85 | intervention=12.5%
```

### 3. Debug Logging Made Optional
Added `debug_ik: bool = False` config option to SO101FollowerEndEffectorConfig. All verbose IK/motor logging now only shows when `debug_ik: true`.

## Config Changes Summary

```json
{
    "target_entropy": -2.0,
    "temperature_lr": 0.0003,
    "action_scale": 0.02
}
```

## Monitoring

Watch for:
- α (entropy coefficient) should rise from 0.03 toward 0.1-0.3
- Policy actions should become more varied (not all near-zero)
- Robot should start exploring different positions

## References

- Luo et al. 2024, "Precise and Dexterous Robotic Manipulation via Human-in-the-Loop Reinforcement Learning"
- SAC paper: Haarnoja et al., "Soft Actor-Critic: Off-Policy Maximum Entropy Deep Reinforcement Learning"
