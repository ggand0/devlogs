# Devlog 023: DrQ-v2 v19 Reward Training Results

**Date:** 2026-01-13

## Training Run

- **Run:** `runs/drqv2/20260112_215549`
- **Reward:** v19 (image-based DrQ-v2)
- **Config:** 16 envs, batch 256, 84x84 RGB, frame_stack=3
- **Exploration:** stddev linear(1.0→0.1, 500k steps)

## Evaluation Results

| Checkpoint | Mean Reward | Max Cube Z | Success Rate |
|------------|-------------|------------|--------------|
| 100k | 342.86 ±16 | 0.019m | 0% |
| 200k | 315.30 ±13 | 0.014m | 0% |
| 300k | 343.78 ±12 | 0.021m | 0% |
| **500k** | **655.68 ±244** | **0.136m** | 0% |
| 800k | 4.26 ±1 | 0.013m | 0% |

## Analysis

### 500k is Peak Performance

The 500k checkpoint achieved significant lifting:
- Episode 0: reward=865.75, max_z=**0.175m** (17.5cm)
- Episode 1: reward=681.91, max_z=**0.172m**
- Episode 3: reward=956.25, max_z=**0.175m**
- Episode 4: reward=483.77, max_z=**0.144m**

Cube is lifted well above the 8cm success threshold. However, success rate shows 0% because the task requires **holding for 150 steps** (3 seconds at 50Hz), which the agent hasn't learned yet.

### 800k Collapse

Same regression pattern as v11:
- Policy collapsed to reward ~4
- Agent moves away from cube
- Max cube Z dropped to 1.3cm (resting height)

### Comparison: v11 vs v19

| Metric | v11 @ 500k | v19 @ 500k |
|--------|------------|------------|
| Mean Reward | 16.19 | **655.68** |
| Max Cube Z | 0.019m | **0.136m** |
| Behavior | Moves away | Grasps & lifts |

v19 is clearly working - the agent learned to grasp and lift. The issue is training stability after 500k steps.

## Observations from Videos

### 100k-300k
- Agent approaches cube consistently
- Closes gripper near cube
- No lifting yet

### 500k (Peak)
- Grasps cube firmly
- Lifts to 14-17cm
- Sometimes drops after lifting
- 4/5 episodes show successful grasping and lifting

### 800k (Collapsed)
- Agent moves away from cube
- No approach behavior
- Policy completely degenerated

## Hypotheses for Collapse

1. **Exploration noise decay**: At 500k, stddev=0.1 (minimum). Policy may have overfit to specific behaviors.

2. **Replay buffer staleness**: 100k buffer fills with old transitions. As policy improves then degrades, buffer contains mixed quality data.

3. **Value function divergence**: Q-values may have grown unstable, causing actor to learn incorrect policy.

4. **Lack of hold reward signal**: Agent never learned to hold because it rarely achieved it during exploration. The hold_count bonus kicks in too late.

## Next Steps

1. **Use 500k checkpoint for sim2real testing** - It's actually working
2. **Investigate training stability**:
   - Reduce hold_steps requirement (150 → 50?)
   - Add early termination on success
   - Try different exploration schedules
3. **Consider SAC instead of DrQ-v2** - SAC may be more stable for longer training

## Files

- Checkpoints: `runs/drqv2/20260112_215549/snapshots/`
- Eval videos: `runs/drqv2/20260112_215549/eval_{100k,200k,300k,500k,800k}/`
