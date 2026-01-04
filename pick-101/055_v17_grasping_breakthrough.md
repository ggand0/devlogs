# Devlog 055: v17 Grasping Breakthrough

## Summary

v17 per-finger reach reward achieves grasping at 200k steps - a breakthrough after multiple failed attempts with v13-v16 and bootstrap training.

## Run Details

**Run:** `runs/image_rl/20260104_000827/`
**Config:** `configs/drqv2_lift_s3_v17.yaml`
**Reward:** v17 (moving finger reach with proximity gate)

## Results @ 200k Steps

| Episode | Grasping % | Max Z (m) | Notes |
|---------|------------|-----------|-------|
| 0 | 1.5% | 0.018 | Brief grasp, dropped |
| 1 | **96.5%** | 0.047 | Sustained grasp, lifting |
| 2 | **89.0%** | 0.054 | Near success threshold (0.08) |
| 3 | **97.5%** | 0.037 | Sustained grasp |
| 4 | **97.5%** | 0.018 | Sustained grasp |
| 5 | **96.5%** | 0.034 | Sustained grasp |
| 6 | 0.0% | 0.026 | Failed to grasp |
| 7 | 0.0% | 0.025 | Failed to grasp |
| 8 | 0.5% | 0.026 | Brief grasp |
| 9 | 5.0% | 0.057 | Some lifting |

**Key metrics:**
- 5/10 episodes with >89% grasping
- Gripper state: mean ~0.17-0.19 (closed, threshold <0.25)
- Max lift height: 0.057m (success threshold: 0.08m)

## Comparison with Previous Attempts

| Method | Steps | Grasping % | Outcome |
|--------|-------|------------|---------|
| v13 (pure RL) | 600k | 0% | Hovering, open gripper |
| v14-v16 | 600k | 0% | Same failure mode |
| Bootstrap + v13 | 600k | 0% | Ignored demonstrations |
| **v17** | **200k** | **89-97%** | **Grasping achieved** |

## Why v17 Works

v17 addresses the core exploration problem identified in devlog 054:

1. **Asymmetric gripper**: Standard reach reward only brings static finger close (it's part of gripper frame). Moving finger gets no gradient.

2. **Image-based observation gap**: Unlike state-based RL (devlog 029), image-based RL can't directly see the relationship between closing gripper and grasping.

3. **v17 solution**: Explicit moving finger reach reward with proximity gate
   - Far from cube: standard reach only
   - Close to cube (gripper_reach >= 0.7): blend in moving finger reach
   - Caps at `is_closed` (gripper_state < 0.25)

This gives ~0.4 reward/step difference between open and closed gripper when near cube - enough gradient to learn closing.

## Training Observations

From debug logs at 200k:
```
Episode 1:
  step 0-4: Gripper closes (1.07 -> 0.27)
  step 5: Grasping begins (reward 2.54)
  step 6+: Sustained grasp, lifting attempts
```

The agent learned:
1. Approach cube (standard reach)
2. Close gripper when close (moving finger reach)
3. Maintain grasp (grasp bonus 1.5)
4. Start lifting (lift reward)

## Final Results @ 2M Steps

Training completed after 8 hours. Final evaluation:

| Episode | Grasping % | Max Z (m) | Success |
|---------|------------|-----------|---------|
| 1 | 97.5% | 0.038 | No |
| 2 | 97.5% | 0.041 | No |
| 3 | 97.5% | 0.083 | No (2 steps above threshold) |
| 4 | 97.5% | 0.043 | No |
| 5 | 97.5% | 0.069 | No |
| 6 | 90.5% | **0.150** | **Yes** (158 steps) |
| 7 | 97.5% | 0.037 | No |
| 8 | 97.5% | 0.038 | No |
| 9 | 97.5% | 0.059 | No |
| 10 | 97.5% | 0.044 | No |

**Summary:**
- Success Rate: 10% (1/10)
- Mean Reward: 747.56 ± 100.37
- Grasping: 90-97.5% in all episodes
- Gripper state: mean ~0.17 (closed)

### Key Achievements

1. **Grasping solved**: 97.5% grasping in 9/10 episodes (vs 0% with v13/bootstrap)
2. **Lifting emerged**: All episodes lift to 0.03-0.08m
3. **First success**: Episode 6 lifted to 0.15m and held

### Comparison with Previous Approaches

| Method | Steps | Grasping % | Success Rate |
|--------|-------|------------|--------------|
| v13 (pure RL) | 600k | 0% | 0% |
| v14-v16 | 600k | 0% | 0% |
| Bootstrap + v13 | 600k | 0% | 0% |
| **v17** | **2M** | **97.5%** | **10%** |

### Training Stats

```
Total timesteps: 2,000,000
Training time: 8 hours
Episodes: ~5000
Final ep_rew_mean: 178-238
```

### Behavior Analysis

From debug logs, the learned policy:
1. **Steps 0-4**: Closes gripper (1.07 → 0.10)
2. **Step 5**: First grasp achieved
3. **Steps 5-200**: Maintains grasp, attempts lifting
4. **Lifting plateau**: Most episodes lift to 0.03-0.06m then plateau

### Remaining Issues

1. **Lifting not aggressive enough**: Agent maintains grasp but doesn't push hard on Z
2. **10% success rate**: Needs stronger lift incentive to reach 0.08m consistently
3. **Hold phase**: Success requires holding above 0.08m, agent often hovers just below

### Next Steps

1. Increase lift reward component (current: tanh scaling)
2. Consider stage 2 style lift reward (steeper gradient near threshold)
3. May need curriculum: first learn to lift high, then add hold requirement

## Files

- Eval videos: `runs/image_rl/20260104_000827/eval_2M/`
- Training videos: `runs/image_rl/20260104_000827/eval_videos/`
- Config: `configs/drqv2_lift_s3_v17.yaml`
- v17 implementation: `src/envs/lift_cube.py:_reward_v17()`
- Checkpoint: `runs/image_rl/20260104_000827/snapshots/2000000_snapshot.pt`
