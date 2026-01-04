# Devlog 056: v18 Stronger Lift Incentive

## Problem

v17 achieved 97.5% grasping but only 10% success rate at 2M steps. Analysis of eval logs showed:

- Most episodes plateau at z=0.03-0.04m
- Agent grasps successfully but doesn't push hard on Z
- Max heights: 0.038, 0.041, 0.043, 0.044 in 6/10 episodes

### Root Cause: Weak Lift Gradient

At z=0.04m, the v17 reward breakdown:
- Reach: ~0.95
- Grasp: 1.5
- Lift: `(0.04-0.015)/(0.08-0.015) * 2.0` = 0.77
- Binary (>0.02m): 1.0
- **Total: ~4.22/step**

At z=0.05m:
- Lift: `(0.05-0.015)/(0.08-0.015) * 2.0` = 1.08
- **Total: ~4.53/step**

**Gradient: only +0.31 per 1cm** - not enough to justify drop risk.

### Drop Risk Analysis

If agent drops while lifting:
- One-time drop penalty: -2.0
- Lost grasp bonus: 1.5/step × remaining steps (~150) = -225
- **Total drop cost: ~227+**

For v17, break-even requires <38% drop probability. Agent learned to play it safe.

## Solution: v18 Reward

Two changes to break the 0.04m plateau:

### 1. Doubled Lift Coefficient (2.0 → 4.0)

```python
lift_progress = max(0, cube_z - 0.015) / (self.lift_height - 0.015)
reward += lift_progress * 4.0  # Was 2.0 in v17
```

### 2. Linear Threshold Ramp (0.04m → 0.08m)

```python
if cube_z > 0.04:
    threshold_progress = min(1.0, (cube_z - 0.04) / (self.lift_height - 0.04))
    reward += threshold_progress * 2.0
```

This adds +0.5/cm in the plateau zone (0.04-0.08m).

## Reward Comparison

| Height | v17 Total | v18 Total | Δ |
|--------|-----------|-----------|---|
| 0.02m | 3.6 | 3.9 | +0.3 |
| 0.03m | 3.9 | 4.5 | +0.6 |
| 0.04m | 4.2 | 5.0 | +0.8 |
| 0.05m | 4.5 | 6.0 | +1.5 |
| 0.06m | 4.9 | 6.8 | +1.9 |
| 0.07m | 5.2 | 7.9 | +2.7 |
| 0.08m | 6.0 | 9.0 | +3.0 |

### Gradient Comparison

| Zone | v17 Δ/cm | v18 Δ/cm |
|------|----------|----------|
| 0.03→0.04 | +0.31 | +0.62 |
| 0.04→0.05 | +0.31 | +1.12 |
| 0.05→0.06 | +0.31 | +0.80 |
| 0.06→0.07 | +0.31 | +1.12 |
| 0.07→0.08 | +0.31 | +1.12 |

The 0.04-0.08m zone now has **~3x stronger gradient**.

### Break-Even Drop Probability

- v17: Must have <38% drop rate to justify pushing
- v18: Can tolerate up to ~55% drop rate

## Implementation

### Files Changed

- `src/envs/lift_cube.py`: Added `_reward_v18()`
- `configs/drqv2_lift_s3_v18.yaml`: Training config

### v18 Reward Structure

```
v18 reward = reach + grasp + lift + bonuses

reach: 0-1 (same as v17, per-finger with proximity gate)
grasp: 1.5 (when is_grasping)
lift:
  - continuous: lift_progress * 4.0 (doubled from v17)
  - binary: +1.0 at 0.02m
  - threshold ramp: linear 0→2.0 from 0.04m→0.08m (NEW)
bonuses:
  - target height: +1.0 at 0.08m
  - success: +10.0
penalties:
  - drop: -2.0 (one-time)
  - push-down: -50*(0.01-z) when z<0.01
  - action rate: -0.02*||Δaction||² during hold phase
```

## Training Command

```bash
MUJOCO_GL=egl uv run python src/training/train_image_rl.py \
    --config configs/drqv2_lift_s3_v18.yaml
```

## Expected Outcome

- Should break the 0.04m plateau
- Agent has stronger incentive to push past 0.06m
- Target: >50% success rate at 2M steps

## Status

Ready for training.
