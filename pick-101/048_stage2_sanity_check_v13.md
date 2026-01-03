# Stage 2 Sanity Check with v13 Reward

**Date:** 2025-01-03
**Status:** In Progress (500k/1.5M)

## Objective

Verify that the agent can learn to grasp and lift with the new camera calibration by using Stage 2 curriculum (cube starts in closed gripper at table level).

## Configuration

- **Config:** `configs/drqv2_lift_s2_sanity.yaml`
- **Curriculum Stage:** 2 (cube in gripper at table)
- **Reward Version:** v13
- **Run Directory:** `runs/image_rl/20260103_072440/`

## Training Progress

### At 500k steps

| Metric | Value |
|--------|-------|
| ep_rew_mean | ~206 |
| success_rate | 0% |
| Peak reward | 206.4 |

## Evaluation Results

### 200k Snapshot
- Mean Reward: 152.49 ± 3.80
- Success Rate: 0%
- Grasping: 0-16% of steps
- Gripper mean: 0.5-1.2 (too open)

### 500k Snapshot
- Mean Reward: 165.10 ± 8.25
- Success Rate: 0%
- Grasping: 4-30% of steps
- Gripper mean: 0.4-1.0

**Key observation:** 500k video shows two-finger contact with the cube. The agent is learning to grasp but not yet lifting.

## Analysis

1. **Progress confirmed:** Agent achieves two-finger grasping at 500k
2. **Cube not lifting:** Max cube Z ~0.025m (needs 0.10m for success)
3. **Camera works:** New calibration is learnable - agent can see and touch cube
4. **More training needed:** 500k may not be enough for Stage 2

## Next Steps

Resume training to 1.5M steps to see if lift behavior emerges.

## Files

- Config: `configs/drqv2_lift_s2_sanity.yaml`
- Run: `runs/image_rl/20260103_072440/`
- Eval videos: `runs/image_rl/20260103_072440/eval_200k/`, `eval_500k/`
