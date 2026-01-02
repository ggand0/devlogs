# Devlog 026: V11 Action Rate Penalty Experiments

## Overview

Testing action rate penalty to reduce twitching/oscillation observed in v10. Based on research documented in devlog 025.

## Experiment 1: V11 with λ=0.05

**Run**: `runs/lift_curriculum_s1/20251229_182729/`
**Eval video**: `runs/lift_curriculum_s1/20251229_182729/eval_200000_combined.mp4`

### Reward Structure

v11 = v10 + action rate penalty + aligned target bonus (z > 0.08)

| Component | Condition | Value |
|-----------|-----------|-------|
| Reach | Always | `1.0 - tanh(10 * distance)` |
| Push-down penalty | `cube_z < 0.01` | `-(0.01 - z) * 50` |
| Drop penalty | Lost grasp | `-2.0` |
| Grasp bonus | Grasping | `+0.25` |
| Continuous lift | Grasping | `lift_progress * 2.0` |
| Binary lift | `cube_z > 0.02` | `+1.0` |
| Target bonus | `z > 0.08` | `+1.0` |
| **Action rate penalty** | Always | `-0.05 * ||a_t - a_{t-1}||²` |
| Success | Held at target | `+10.0` |

### Results

- **Training**: 200k steps
- **Rollout success**: 13%
- **Final eval**: 0% success rate
- **Behavior**: Agent stuck at z=0.054m, not lifting at all
- **Mean reward**: ~428

| Episode | Final z | Success |
|---------|---------|---------|
| 1-5 | 0.054 | ❌ |

### Analysis

**Action smoothness worked**: Actions became extremely stable - almost identical across all steps:
```
step=110: z=0.0542, act=[+0.54,-0.00,+0.00,-0.28], grasp=True
step=111: z=0.0542, act=[+0.54,-0.00,+0.00,-0.28], grasp=True
step=112: z=0.0542, act=[+0.54,-0.00,+0.00,-0.28], grasp=True
...
```

**But lifting stopped**: The penalty coefficient (0.05) was too strong. The agent learned to hold still rather than lift because any action change incurs penalty. At z=0.054m with stable actions, the agent avoids the penalty while still getting grasp bonus and reach reward.

**Problem**: Unconditional action penalty discourages ALL action changes, including the ones needed to lift the cube.

## Solution: Conditional Penalty

Only apply action rate penalty when cube is already lifted (z > 0.06). This allows:
- Free lifting below 0.06m (no penalty)
- Smooth holding above 0.06m (penalty active)

Also reduced coefficient from 0.05 to 0.01 for less aggressive smoothing.

```python
# Only penalize when near target height
if action is not None and cube_z > 0.06:
    action_delta = action - self._prev_action
    action_penalty = 0.01 * np.sum(action_delta**2)
    reward -= action_penalty
```

## Experiment 2: V11 with Conditional Penalty (z > 0.06, λ=0.01)

**Run**: `runs/lift_curriculum_s1/20251229_210901/`
**Eval video**: `runs/lift_curriculum_s1/20251229_210901/eval_200000_combined.mp4`

### Reward Structure

Same as Experiment 1, but action penalty only applies when z > 0.06:

| Component | Condition | Value |
|-----------|-----------|-------|
| Action rate penalty | `z > 0.06` | `-0.01 * \|\|a_t - a_{t-1}\|\|²` |

### Results

- **Training**: 200k steps
- **Rollout success**: 11%
- **Final eval**: 60% success rate (6/10 episodes)
- **Behavior**: Successful lifts reach z=0.089-0.108m, failures stuck at z=0.062-0.073m
- **Mean reward**: ~355 (high variance due to early termination on success)

| Episode | Final z | Success |
|---------|---------|---------|
| 1 | 0.107 | ✅ |
| 2 | 0.096 | ✅ |
| 3 | 0.062 | ❌ |
| 4 | 0.100 | ✅ |
| 5 | 0.108 | ✅ |
| 6 | 0.089 | ✅ |
| 7 | 0.073 | ❌ |
| 8 | 0.095 | ✅ |
| 9 | 0.066 | ❌ |
| 10 | 0.067 | ❌ |

### Analysis

**First successful lift policy!** The conditional penalty worked:
- Actions are smooth above z=0.06m
- Agent can still lift freely below 0.06m
- Successful episodes lift well past target (0.089-0.108m) and hold stable

**Remaining failure mode**: Some episodes get stuck at z=0.062-0.073m. The penalty kicks in right at z=0.06 and sometimes causes the agent to park there rather than pushing through to 0.08m.

**Potential improvements**:
1. Raise conditional threshold to z > 0.07 (closer to target)
2. Further reduce penalty coefficient
3. Increase target bonus to incentivize pushing through the penalty zone

## Experiment 3: 500k Training with 3-Second Hold Requirement

**Run**: `runs/lift_curriculum_s1/20251229_224741/`
**Eval video**: `runs/lift_curriculum_s1/20251229_224741/eval_500000_combined.mp4`

### Config Changes

- `timesteps: 500000` (up from 200k)
- `hold_steps: 150` (~3 seconds at 50Hz, up from 10)
- `max_episode_steps: 300` (up from 100)

### Results

- **Training**: 500k steps
- **Rollout success**: 62%
- **Final eval**: 100% success rate (10/10 episodes)
- **Behavior**: Agent lifts aggressively to z=0.225-0.237m, holds for 3 seconds
- **Mean reward**: ~43

| Episode | Final z | Success |
|---------|---------|---------|
| 1 | 0.227 | ✅ |
| 2 | 0.233 | ✅ |
| 3 | 0.237 | ✅ |
| 4 | 0.229 | ✅ |
| 5 | 0.225 | ✅ |
| 6 | 0.236 | ✅ |
| 7 | 0.232 | ✅ |
| 8 | 0.236 | ✅ |
| 9 | 0.225 | ✅ |
| 10 | 0.231 | ✅ |

### Analysis

**Robust lift policy achieved!** The longer hold requirement (3 seconds) and more training produced a robust policy:
- Agent lifts quickly (z > 0.08 by step 5-7)
- Overshoots to z=0.22-0.24m (well above target)
- Holds stable for 150 steps
- 100% success across varied initial conditions (±2cm cube position randomization)

The aggressive z-action (close to +1.0) shows the agent learned to push past the action rate penalty zone quickly rather than getting stuck.

## Key Learnings

1. **Action rate penalty works for smoothness**: Completely eliminated twitching when λ=0.05
2. **But can prevent task completion**: If penalty is too strong or unconditional, agent prefers doing nothing
3. **Conditional penalties are better**: Apply smoothness penalty only where it matters (at target height)
4. **Coefficient tuning matters**: 0.05 too strong, 0.01 with conditional activation achieves 60% success
5. **Longer hold requirement helps**: 3-second hold (150 steps) forces more robust holding behavior
6. **More training helps**: 500k steps achieved 100% eval success vs 60% at 200k
