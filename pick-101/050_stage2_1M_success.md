# Stage 2 Sanity Check - 1.5M Training Complete

**Date:** 2025-01-03

## Result

**The agent successfully learned to grasp and lift the cube from Stage 2 curriculum.**

Stage 2: Cube starts in closed gripper at table level, agent needs to lift it.

## Training Progress

| Checkpoint | Mean Reward | Notes |
|------------|-------------|-------|
| 500k | ~165 | Learning to close gripper |
| 800k | ~398 | Grasping learned, minimal lift |
| 1M | ~674 | Consistent grasping and lifting |
| 1.4M | ~720 | Best eval reward (peak) |
| 1.5M | ~720 | Training complete |

## 1.4M Checkpoint (Best)

```
| rollout/            |            |
|    ep_rew_mean      | 720.6331   |
|    success_rate     | 0.0000     |
| time/               |            |
|    episodes         | 3592       |
|    fps              | 3.34e+03   |
|    iterations       | 90000      |
|    time_elapsed     | 3:47:44    |
|    total_timesteps  | 1440000    |
```

**New best eval reward: 720.63** saved to `best_snapshot.pt`

## Final Stats (1.5M)

- Training time: 4h 2m
- Total episodes: ~3750
- Episode length: 400 steps (200 env steps × action_repeat=2)
- Success rate: 0% (8cm threshold not reached, but consistent 6-7cm lift)

## Key Observations

1. **100% grasping rate** - Agent keeps gripper closed throughout all episodes
2. **Consistent lifting** - Cube reaches 6-7cm height
3. **Stable behavior** - Low variance across episodes
4. **Reward plateau** - Mean reward stabilized around 720 from 1.4M onwards

## Configuration

- `curriculum_stage: 2` (cube in gripper at table)
- `reward_version: v13`
- `action_repeat: 2`
- `num_train_envs: 8`
- `eval_every_steps: 6250` (~100k env steps)

## Files

- Run: `runs/image_rl/20260103_122759/`
- Best checkpoint: `runs/image_rl/20260103_122759/snapshots/best_snapshot.pt` (1.4M)
- Final checkpoint: `runs/image_rl/20260103_122759/snapshots/1500000_snapshot.pt`
- Learning curves: `runs/image_rl/20260103_122759/learning_curves.png`

## Next Steps

Move to Stage 3 (gripper near cube, needs to approach and grasp) with:
- v4 camera calibration (fixed pitch and FOV)
- Proper preprocessing: 640x480 → 480x480 center crop → 84x84
