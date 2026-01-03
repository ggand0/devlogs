# Stage 3 Bootstrap Experiment

## Background

Stage 3 training with v4 camera + v13 reward shows same single-finger behavior at 600k steps:
- Run: `runs/image_rl/20260103_183217/`
- Agent keeps gripper open, only single-finger contact
- No gradient toward closing gripper (same issue as v14-v16)

## Root Cause

From devlog 047 analysis:
- Random exploration rarely produces close + two-finger contact simultaneously
- Reach reward (~0.9/step) is maximized without discovering grasp reward
- No intermediate reward for closing gripper when near cube

## Solution: Bootstrap Training

Following QT-Opt approach - seed replay buffer with scripted grasp trajectories.

**Scripted Policy Stats:**
- Success rate: ~80%
- Phases: descend (25 steps) -> close (20 steps) -> lift -> hold (10 steps)
- Episodes terminate on success (~60 steps) or run full 200 steps

## Experiment Plan

1. Collect 1000 scripted trajectories
2. Seed replay buffer with successful grasp demonstrations
3. Train with same config (v13 reward, stage 3)

## Commands

```bash
# Step 1: Collect scripted trajectories
MUJOCO_GL=egl uv run python scripts/collect_scripted_grasps.py \
    --episodes 1000 \
    --output runs/bootstrap/scripted_grasps_1k.pkl \
    --curriculum_stage 3

# Step 2: Train with bootstrap
MUJOCO_GL=egl uv run python scripts/train_with_bootstrap.py \
    --config configs/drqv2_lift_s3_v13.yaml \
    --bootstrap runs/bootstrap/scripted_grasps_1k.pkl
```

## Expected Outcome

- Replay buffer starts with ~60k successful grasp transitions
- Agent should discover grasp reward early through replay
- Should overcome exploration barrier that blocked v13-v16 rewards

## Comparison

| Experiment | Method | Expected Result |
|------------|--------|-----------------|
| 20260103_183217 | Pure RL (v13) | Single-finger, no grasp |
| This exp | Bootstrap + RL | Should learn grasping |

## Results: Bootstrap Failed

**Run:** `runs/image_rl_bootstrap/20260103_211328/`

Bootstrap training converged to the same failure mode at 600k steps:

### Eval @ 600k Steps
```
Episode 0:
  Steps: 200
  Total reward: 154.295
  Cube Z: min=0.0150, max=0.0204, final=0.0155
  Grasping: 0 steps (0.0%)
  Gripper: min=1.065, max=1.095, mean=1.080 (closed<0.25)
```

### Key Metrics
| Metric | Value |
|--------|-------|
| Train Success Rate | 0% (flat) |
| Eval Reward | ~62.89 |
| Gripper State | 1.08 (wide open) |
| Grasping Steps | 0% |

### Learning Curves
- Train Episode Reward: ~161 (saturated)
- Train Success Rate: flat at 0%
- Q-Value: increasing to ~50.83 (overestimation)

### Analysis

Bootstrap seeding did not solve the exploration problem:

1. **Same local optimum**: Agent still maximizes reach reward without closing gripper
2. **Ignored demonstrations**: Despite 60k+ successful grasp transitions in buffer, agent converges to hovering policy
3. **Reward structure issue**: The fundamental problem is that reach reward (~0.9/step) dominates, making grasp exploration not worth the risk

### Why Bootstrap Failed

The QT-Opt paper used sparse rewards (+1 success, -0.05/step), so the only way to get positive reward was through successful grasps. Our dense reach reward creates a local optimum that bootstrap seeding cannot escape.

**Possible fixes:**
1. Use sparse rewards (but loses gradient for reaching)
2. Per-finger reach reward (v17) - gives gradient for both fingers independently
3. Higher grasp bonus to outweigh reach (tried v16, didn't work)

### Next Step

Implemented v17 per-finger reach reward - see devlog 054.
