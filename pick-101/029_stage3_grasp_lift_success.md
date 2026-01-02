# Devlog 029: Stage 3 Grasp + Lift - Success from Scratch

## Overview

After failed curriculum transfer attempts (see devlog 027, 028), trained stage 3 (grasp + lift) from scratch. Achieved 100% success rate.

## Configuration

**Run**: `runs/lift_curriculum_s3/20251230_135113/`

```yaml
training:
  timesteps: 1000000
  seed: 42

env:
  max_episode_steps: 300
  action_scale: 0.02
  lift_height: 0.08
  hold_steps: 150
  reward_version: "v11"
  curriculum_stage: 3  # Gripper near cube, open
  lock_wrist: true
```

## Results

### Training Progress

| Checkpoint | Eval Success Rate |
|------------|-------------------|
| 100k | ~20% |
| 500k | ~60% |
| 1M | 80% (stochastic) |

### Final Evaluation (Deterministic)

**Success rate: 100% (10/10)**

| Episode | Final z | Steps to Success |
|---------|---------|------------------|
| 1 | 0.146m | 32 |
| 2 | 0.131m | 29 |
| 3 | 0.171m | ~30 |
| 4 | 0.209m | ~35 |
| 5 | 0.235m | 30 |
| 6 | 0.257m | 43 |
| 7-10 | 0.15-0.18m | 25-30 |

Mean reward: 67.80 ± 13.95

### Emergent Behavior: "Bump and Grab"

Interesting strategy the agent discovered:

```
step=  0-10: Gripper moves erratically, bumps cube
step= 10-15: Cube rotates/shifts position
step= 15-20: Gripper closes, grasps cube
step= 20-30: Lifts to target height
step= 30+: Holds for 150 steps → success
```

The agent learned to "nudge" the cube into a graspable orientation before closing the gripper. This wasn't explicitly rewarded - it emerged from the dense reward structure encouraging proximity and eventual grasp.

Example from Episode 1:
```
step=  0: z=0.0150, act=[+0.94,+0.95,-0.78,+0.98], grasp=False  # wild initial movement
step= 10: z=0.0150, act=[-0.32,+0.72,-0.91,-0.92], grasp=False  # approaching
step= 16: z=0.0203, act=[+0.92,+0.99,-0.87,-0.99], grasp=True   # first grasp!
step= 17: z=0.0232, act=[+0.16,+0.78,+1.00,-0.99], grasp=True   # lifting
step= 32: z=0.1458, r=+12.168, grasp=True                        # SUCCESS
```

## Comparison: Curriculum vs From Scratch

| Approach | Training Time | Final Success |
|----------|---------------|---------------|
| Stage 1 only | 500k steps | 100% (hold task) |
| Stage 1 → Stage 3 transfer | Failed | 0% |
| Stage 3 from scratch | 1M steps | 100% |

The curriculum transfer failed due to large distributional gap between stages (see devlog 028). Training from scratch with 2x the steps achieved the full task.

## Key Observations

1. **Grasp timing varies**: Agent grasps between step 12-20 depending on initial cube position
2. **Overshoots target**: Lifts to 0.13-0.26m (target is 0.08m) - aggressive lifting strategy
3. **Stable hold**: Once lifted, holds reliably for 150 steps
4. **Robust to position variation**: Works across ±2cm cube position randomization

## Training Stats

- **Duration**: ~4 hours on RTX 4090
- **Final entropy**: 0.00518 (low but not collapsed)
- **Actor loss**: -6.8
- **Critic loss**: 0.0378

## Files

- Model: `runs/lift_curriculum_s3/20251230_135113/checkpoints/sac_lift_1000000_steps.zip`
- Normalization: `runs/lift_curriculum_s3/20251230_135113/vec_normalize.pkl`
- Video: `runs/lift_curriculum_s3/20251230_135113/eval_1000000_combined.mp4`

## Demo

X post: https://x.com/gtgando/status/2005970149091004424

## Next Steps

1. Test with larger cube position randomization
2. Try different initial gripper positions (not just "near cube")
3. Add obstacle avoidance
4. Transfer to real robot
