# Devlog 028: Curriculum Learning Transfer Research

## Overview

Research into why curriculum transfer from stage 1 → stage 3 failed, and proper methods for curriculum learning with distribution shifts.

## The Problem

Stage 1 policy (100% success at holding lifted cube) completely failed when transferred to stage 3 (grasp + lift from ground):

| Stage | Initial State | Task | Learned Behavior |
|-------|---------------|------|------------------|
| 1 | cube at z=0.22m, gripper closed | Hold | "Don't move, stay still" |
| 3 | cube at z=0.015m, gripper open | Grasp + Lift | Requires active movement |

Transfer result: 0% success, agent moves away from cube, entropy collapsed to 0.0008.

## Root Cause: Distributional Gap Too Large

The VecNormalize handling was a red herring. Fresh normalization is correct for curriculum transfer. The real problem: **the behavioral gap between stages is too large**.

Stage 1 → Stage 3 skips the intermediate difficulty states that curriculum learning requires.

## Research Findings

### Reverse Curriculum Generation (Florensa et al., 2017)

**Paper**: [Reverse Curriculum Generation for Reinforcement Learning](https://arxiv.org/abs/1707.05300)

Key insight: Train from goal states outward, progressively expanding to harder starting positions.

The algorithm identifies **Starts of Intermediate Difficulty (SoID)** - positions where the current policy succeeds 10-90% of the time. Training only on these positions provides consistent learning signal.

**Why our transfer failed**: Stage 1 policy has ~0% success on stage 3's initial state distribution. There's no overlap in the "intermediate difficulty" zone.

### GRADIENT: Optimal Transport for Curriculum (NeurIPS 2022)

**Paper**: [Curriculum RL using Optimal Transport via Gradual Domain Adaptation](https://arxiv.org/abs/2210.10195)

Formulates curriculum as interpolations between source and target task distributions using **Wasserstein barycenters** - geometric interpolations that create smooth transitions.

**Key principle**: Break large distributional shifts into smaller, manageable steps.

### Progressive Neural Networks (Rusu et al., 2016)

**Paper**: [Sim-to-Real Robot Learning from Pixels with Progressive Nets](https://arxiv.org/abs/1610.04286)

Alternative architecture approach:
- Freeze source network columns
- Add new columns with lateral connections for target task
- Prevents catastrophic forgetting while enabling transfer

**Trade-off**: Requires architecture changes, increases parameter count.

### Progressive Transfer Learning for Manipulation (2023)

**Paper**: [Progressive Transfer Learning for Dexterous In-Hand Manipulation](https://arxiv.org/abs/2304.09526)

Uses progressive nets + selective experience replay:
- Prioritize trajectories based on reward, smoothness, dynamics similarity
- Gradually shift from source to target data
- Achieved 95% training time reduction vs. from-scratch

## Correct Curriculum Design

For our lift task, proper reverse curriculum:

| Stage | Initial State | Task | Transfers From |
|-------|---------------|------|----------------|
| 1 | cube at z=0.22m, gripper closed | Hold | - |
| **2** | cube at z=0.015m, gripper closed | Lift + Hold | Stage 1 |
| 3 | cube at z=0.015m, gripper open | Grasp + Lift + Hold | Stage 2 |

**Stage 2 bridges the gap**:
- Same cube position as stage 3 (observation distribution overlap)
- Same lift behavior as stage 1 (policy transfer works)
- Only adds the lift component, not grasp

## Why VecNormalize Wasn't the Issue

| Approach | Result | Reason |
|----------|--------|--------|
| Fresh VecNormalize (original) | 0% success | Policy weights encode wrong behavior |
| Load stage 1 VecNormalize | 0% success | Observations misscaled + wrong behavior |
| Fresh VecNormalize (confirmed) | Correct approach | Normalization adapts, but policy still wrong |

The normalization stats converge within ~1000 steps. The policy weights are the problem - they encode "hold still" which is wrong for "approach and grasp".

## Practical Recommendations

1. **Add Stage 2**: Create intermediate curriculum stage with cube grasped at ground level
2. **Smaller steps**: Each stage should have 10-90% overlap with next stage
3. **Fresh normalization**: Always use fresh VecNormalize for curriculum transfer (confirmed correct)
4. **Consider Progressive Nets**: If architectural changes acceptable, freeze source + add capacity

## Decision

Training stage 3 from scratch while implementing stage 2 for future curriculum experiments.

## References

- [Reverse Curriculum Generation (BAIR Blog)](https://bair.berkeley.edu/blog/2017/12/20/reverse-curriculum/)
- [Reverse Curriculum Generation Paper](https://arxiv.org/abs/1707.05300)
- [GRADIENT: Optimal Transport for Curriculum RL](https://arxiv.org/abs/2210.10195)
- [Progressive Neural Networks](https://arxiv.org/abs/1610.04286)
- [Progressive Transfer Learning for Manipulation](https://arxiv.org/abs/2304.09526)
- [SB3 VecNormalize Documentation](https://stable-baselines3.readthedocs.io/en/master/guide/vec_envs.html)
