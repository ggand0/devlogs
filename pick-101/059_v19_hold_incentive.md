# Devlog 057: v19 Hold Count Bonus

## Problem

v18 lifts higher (0.06-0.10m vs v17's 0.03-0.04m) but regressed to 0% success rate.

**Root cause**: Success requires 10 consecutive steps above 0.08m (`hold_count >= 10`). v18's agent oscillates above/below threshold, resetting the counter each time. Episode 7 had 63 steps above 0.08m but they weren't consecutive.

The reward structure incentivizes reaching height but not staying there. Once at 0.08m, there's no additional reward for maintaining position vs. moving around.

## Solution: Hold Count Bonus

Add escalating reward for consecutive steps at target height:

```python
if cube_z > self.lift_height:
    reward += 1.0  # existing target bonus
    reward += 0.5 * hold_count  # NEW: 0.5, 1.0, 1.5, ... 5.0
```

### Reward Analysis

| Hold Step | Bonus | Cumulative |
|-----------|-------|------------|
| 1 | +0.5 | 0.5 |
| 2 | +1.0 | 1.5 |
| 3 | +1.5 | 3.0 |
| 5 | +2.5 | 7.5 |
| 10 | +5.0 | 27.5 |

**Key insight**: At step 9, dipping would forfeit the +4.5 bonus for that step AND reset cumulative progress. This creates strong incentive to stabilize once near success.

## Implementation

### Files Changed

- `src/envs/lift_cube.py`: Added `_reward_v19()`
- `configs/drqv2_lift_s3_v19.yaml`: Training config

### v19 Reward Structure

Same as v18, plus:
- Hold count bonus: `+0.5 * hold_count` when `cube_z > 0.08m`

## Training Command

```bash
MUJOCO_GL=egl uv run python src/training/train_image_rl.py \
    --config configs/drqv2_lift_s3_v19.yaml
```

## Expected Outcome

- Should maintain v18's lift ability (reaching 0.06-0.10m)
- Reduced oscillation at target height
- Target: >30% success rate at 2M steps

## Training Results (2M Steps)

### Evaluation: 10 Episodes

| Metric | Value |
|--------|-------|
| Success Rate | **100%** |
| Mean Reward | 190.97 ± 6.31 |
| Episode Length | 19-22 steps |

### Per-Episode Breakdown

| Episode | Steps | Max Height | Steps >0.08m | Success |
|---------|-------|------------|--------------|---------|
| 0 | 21 | 0.119m | 10 | ✓ |
| 1 | 21 | 0.103m | 10 | ✓ |
| 2 | 22 | 0.107m | 10 | ✓ |
| 3 | 20 | 0.098m | 10 | ✓ |
| 4 | 22 | 0.108m | 10 | ✓ |
| 5 | 20 | 0.122m | 10 | ✓ |
| 6 | 21 | 0.113m | 10 | ✓ |
| 7 | 19 | 0.111m | 10 | ✓ |
| 8 | 20 | 0.108m | 10 | ✓ |
| 9 | 20 | 0.104m | 10 | ✓ |

### Comparison Across Versions

| Metric | v17 | v18 | v19 |
|--------|-----|-----|-----|
| Success Rate | 10% | 0% | **100%** |
| Episode Length | 200 | 200 | 19-22 |
| Max Height | 0.044m | 0.100m | 0.122m |
| Steps >0.08m | rare | scattered | exactly 10 |

### Key Finding

The hold count bonus completely solved the oscillation problem. Every episode:
- Reached exactly 10 consecutive steps above 0.08m
- Stabilized at height instead of oscillating
- Completed in ~20 steps (vs 200 step timeouts)

The lower mean reward (191 vs 1362) is expected - episodes terminate on success instead of accumulating rewards over 200 steps.

## X Post

https://x.com/gtgando/status/2008205321937060211

## Status

**SOLVED.** v19 achieves 100% success rate on image-based lift task at 2M steps.
