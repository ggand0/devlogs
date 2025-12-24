# Devlog 016: Curriculum Training v8 Results and v9 Reward

## v8 Training Results (500k steps)

**Config:** `configs/lift_curriculum_s1.yaml` with reward_version v8

```
Eval num_timesteps=500000, episode_reward=178.20 +/- 42.30
Episode length: 100.00 +/- 0.00
Success rate: 0.00%
```

### Eval Video Analysis

Ran `eval_cartesian.py` with 3 episodes:

| Episode | Reward | Success | Max cube_z | Notes |
|---------|--------|---------|------------|-------|
| 1 | 431.52 | No | 0.047 | Maintained grasp, lifted halfway to target (0.08) |
| 2 | 133.62 | No | 0.015 | Dropped cube early, never recovered |
| 3 | 425.28 | No | 0.021 | Maintained grasp, barely lifted |

### Problem Identified

v8 reward structure:
- Reach reward: ~1.0 when close
- Grasp bonus: +0.25 per step
- Binary lift bonus: +1.0 when cube_z > 0.02
- No gradient for lifting higher

Episode 1 reached z=0.047 (59% to target) but got the same +1.0 lift bonus as z=0.021. No incentive to lift higher.

## v9 Reward Design

Added continuous lift gradient when grasping:

```python
if is_grasping:
    reward += 0.25  # Base grasp bonus

    # NEW: Continuous lift reward
    lift_progress = max(0, cube_z - 0.015) / (self.lift_height - 0.015)
    reward += lift_progress * 2.0  # Up to +2.0 at target height
```

**Reward comparison at different heights (when grasping):**

| cube_z | v8 reward | v9 reward | Difference |
|--------|-----------|-----------|------------|
| 0.015 (table) | 1.25 | 1.25 | +0.0 |
| 0.02 | 2.25 | 2.40 | +0.15 |
| 0.047 (ep1 max) | 2.25 | 3.23 | +0.98 |
| 0.08 (target) | 2.25 | 4.25 | +2.0 |

v9 creates a smooth gradient encouraging the agent to lift higher, not just clear the 0.02 threshold.

## v9 Training Results (500k steps)

```
Eval num_timesteps=500000, episode_reward=236.90 +/- 125.22
Episode length: 100.00 +/- 0.00
Success rate: 0.00%
```

### Training Trajectory

Early promise at ~38k steps (8% through training):
- Rollout success_rate: 3-4%
- Actor_loss: -12.9 (finding good actions)

Policy collapsed by end:
- Rollout success_rate: 0%
- Actor_loss: -4.0 (back to local optimum)

### Eval Video Analysis

| Episode | Reward | Success | Max cube_z | Final cube_z | % of target |
|---------|--------|---------|------------|--------------|-------------|
| 1 | 427.50 | No | 0.069 | 0.056 | 86% / 70% |
| 2 | 398.14 | No | 0.024 | 0.023 | 30% |
| 3 | 405.75 | No | 0.062 | 0.059 | 78% / 74% |

**Better than v8:** v9 reached z=0.069 vs v8's z=0.047. The continuous lift gradient helped.

**Observed behavior:**
- Agent learned lift-and-hover behavior
- Successfully re-grabs cube after dropping in some episodes
- Plateaus at ~70% of target height instead of pushing to 0.08

### Why 0% Success Despite Good Behavior?

The lift gradient (+2.0 max) is still dominated by:
- Reach reward: ~1.0 per step (always on when close)
- Grasp bonus: +0.25 per step

Agent finds local optimum: maintain grasp + hover at comfortable height = consistent reward without risk of dropping.

## v10 Reward Design

Changes from v9:
1. **No reach reward when grasping** - redundant, dominates signal
2. **Stronger lift gradient** - 5.0 instead of 2.0
3. **Stronger drop penalty** - 5.0 instead of 2.0

```python
if is_grasping:
    reward += 0.1  # Small grasp maintenance
    lift_progress = max(0, cube_z - 0.015) / (self.lift_height - 0.015)
    reward += lift_progress * 5.0  # Up to +5.0 at target height
else:
    # Only reach reward when not grasping (recovery)
    reach_reward = 1.0 - np.tanh(10.0 * gripper_to_cube)
    reward += reach_reward
```

## Files Changed

- `envs/lift_cube.py` - Added `_reward_v9()` and `_reward_v10()` functions
- `configs/lift_curriculum_s1.yaml` - reward_version: v8 -> v9 -> v10
