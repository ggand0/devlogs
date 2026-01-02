# Devlog 027: VecNormalize Curriculum Transfer

## Overview

Investigation into curriculum learning transfer between stages with different observation distributions.

## Initial Bug: Missing VecNormalize Stats

First attempt at curriculum transfer (stage 1 → stage 3) failed because VecNormalize stats weren't being transferred with pretrained weights. Fixed by loading `vec_normalize.pkl` alongside the model checkpoint.

## Second Bug: Wrong Stats for Different Distribution

After fixing the first bug, transfer STILL failed. The problem: loading stage 1 normalization stats for stage 3 causes misscaled observations.

Stage 1 normalization was tuned for:
- `cube_z ≈ 0.22m` (lifted)
- Gripper closed, grasping

Stage 3 observations:
- `cube_z = 0.015m` (ground)
- Gripper open, not grasping

When stage 3 observations are normalized using stage 1 stats:
```
normalized_cube_z = (0.015 - 0.22) / std = large negative number
```

The policy receives out-of-distribution inputs and outputs garbage.

## Symptoms

Same as before - agent moves away from cube, never grasps:
```
step=  0: z=0.0150, r=+0.677, act=[-0.98,+0.31,-0.93,-0.99], grasp=False
step=  1: z=0.0150, r=+0.713, act=[-0.94,+0.21,-0.97,-0.49], grasp=False
...
```

Entropy collapsed to 0.000834, indicating the policy stopped exploring.

## The Correct Fix

For curriculum learning between stages with **different observation distributions**:
- **Resume** (same task): Load VecNormalize stats
- **Pretrained** (different task): DON'T load VecNormalize stats, use fresh normalization

```python
vec_normalize_path = None
if args.resume:
    vec_normalize_path = resume_dir / "vec_normalize.pkl"
# Note: pretrained does NOT load VecNormalize - different curriculum stages
# have different observation distributions
```

The policy weights provide useful initialization, but the normalization must adapt to the new observation distribution.

## Proper Curriculum Learning

For curriculum learning to work well with VecNormalize, stages should have **overlapping observation distributions**:

| Stage | Cube Position | Gripper State | Task |
|-------|---------------|---------------|------|
| 1 | z=0.22m (lifted) | Closed, grasping | Hold |
| 2 | z=0.015m (ground) | Closed, grasping | Lift |
| 3 | z=0.015m (ground) | Open, not grasping | Grasp + Lift |

Skipping stage 2 (going directly 1→3) creates a large distributional shift that's hard to bridge.

## Key Learnings

1. **VecNormalize stats must match observation distribution**: Can't use stats from one distribution with observations from another
2. **Fresh normalization for curriculum transfer**: When transferring to a stage with different observations, start fresh VecNormalize
3. **Curriculum stages should overlap**: Gradual transitions work better than large jumps
4. **Entropy collapse indicates problem**: ent_coef dropping to ~0.001 means policy stopped exploring

## Files Changed

- `train_lift.py`: Don't load VecNormalize stats for pretrained (curriculum transfer)
