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

## Training Results (2M Steps)

### Evaluation: 10 Episodes

| Metric | Value |
|--------|-------|
| Success Rate | 0% |
| Mean Reward | 1362.17 ± 284.48 |
| Grasping | 93-97.5% |

### Lift Height Analysis

| Episode | Max Height | Steps >0.06m | Steps >0.08m |
|---------|------------|--------------|--------------|
| 0 | 0.076m | 32 | 0 |
| 1 | 0.080m | 45 | 1 |
| 2 | 0.069m | 18 | 0 |
| 3 | 0.082m | 51 | 12 |
| 4 | 0.091m | 58 | 34 |
| 5 | 0.078m | 41 | 0 |
| 6 | 0.085m | 49 | 21 |
| 7 | 0.100m | 71 | 63 |
| 8 | 0.073m | 25 | 0 |
| 9 | 0.088m | 55 | 28 |

### Comparison vs v17

| Metric | v17 (2M) | v18 (2M) |
|--------|----------|----------|
| Success Rate | 10% | 0% |
| Mean Reward | 747.56 | 1362.17 |
| Max Height | 0.044m | 0.100m |
| Typical Height | 0.03-0.04m | 0.06-0.08m |

### Key Finding

v18 lifts significantly higher but **cannot sustain at target height**. The agent oscillates - goes up, comes back down. Episode 7 had 63 steps above 0.08m and reached 0.10m but still failed.

**Root cause**: No incentive to hold position. Agent optimizes instantaneous reward, not sustained height.

### Next Step

v19 needs a **hold incentive** - reward for maintaining height over time, not just reaching it.

## Artifact Backup

Model weights and eval videos backed up to `/data/ggando/pick-101/runs/image_rl/`:

| Directory | Version | Contents |
|-----------|---------|----------|
| `v17_20260104_000827/` | v17 | 25 snapshots, 30 videos, config |
| `v18_20260104_103333/` | v18 | 25 snapshots, 30 videos, config |

Each ~928MB, total ~1.9GB.

## Status

v18 training complete. Lifts higher but regressed on success rate.
