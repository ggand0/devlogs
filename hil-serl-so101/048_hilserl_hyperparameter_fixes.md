# Devlog 048: HIL-SERL Hyperparameter Fixes

**Date**: 2025-01-25
**Status**: In Progress

## Problem

After 100+ episodes of HIL-SERL training, the robot still exhibits constant shaking and cannot reach the cube without human intervention (~80% intervention rate). The policy appears to be learning from interventions but hasn't generalized to autonomous reaching.

## Root Cause Analysis

Compared our implementation against the [HIL-SERL paper](https://hil-serl.github.io/) and [LeRobot HIL-SERL docs](https://huggingface.co/docs/lerobot/en/hilserl). Found multiple critical hyperparameter mismatches:

### 1. Missing Step Penalty (FIXED)

**Problem**: Reward structure was sparse binary (0 or 1) with no step penalty.

**HIL-SERL uses**:
- Success: **+10**
- Every other step: **-0.05** (step penalty)

The step penalty provides learning signal at every timestep, encouraging:
- Faster task completion
- Discouraging shaking/stalling (accumulates negative reward)
- Smoother trajectories

**Fix applied**:
```python
# Before
reward = 0.0
if success == 1.0:
    reward = 1.0

# After
if success == 1.0:
    reward = 10.0
else:
    reward = -0.05  # Step penalty
```

### 2. UTD Ratio Too Low

**Problem**: `utd_ratio = 2`

**RLPD/HIL-SERL uses**: `utd_ratio = 20`

UTD (Update-to-Data) ratio controls how many gradient updates happen per environment step. With UTD=2, the learner only does 2 updates per sample. With UTD=20, it does 10x more learning per sample.

**Impact**: With UTD=2, the policy learns much slower than it should, requiring many more samples to converge.

**Fix needed**: Change `utd_ratio` from 2 to 20

### 3. Number of Offline Demos

**HIL-SERL paper states**: "20-30 trajectories for initial training"

**Current dataset**: 15 episodes (`gtgando/so101_reach_grasp_cube_reward_v3`)

This might be borderline insufficient. Consider collecting 5-15 more demos.

### 4. Discrete vs Continuous Gripper

**HIL-SERL uses**: Separate DQN critic for discrete gripper (3 states: open/close/stay)

**Our implementation**: Continuous gripper action

This is a design difference, not necessarily wrong, but the original paper found discrete gripper control more effective for manipulation.

### 5. Temperature Too High (FIXED)

**Problem**: `temperature_init = 0.2`

**Berkeley's HIL-SERL uses**: `temperature_init = 0.01` (1e-2)

Verified from `/home/gota/ggando/ml/hil-serl/serl_launcher/serl_launcher/utils/launcher.py` lines 83, 133, 183.
All three agent types (`make_sac_pixel_agent`, `make_sac_pixel_agent_hybrid_single_arm`, `make_sac_pixel_agent_hybrid_dual_arm`) use `temperature_init=1e-2`.

**Impact**: With temperature 20x too high, the policy adds excessive exploration noise, causing shaking and making human interventions ineffective.

**Fix applied**: Change `temperature_init` from 0.2 to 0.01

## Summary of Required Changes

| Parameter | Current | Required | Ratio |
|-----------|---------|----------|-------|
| `temperature_init` | 0.2 | 0.01 | 20x lower ✓ |
| `utd_ratio` | 2 | 20 | 10x higher ✓ |
| Step penalty | None (0) | -0.05 | Added ✓ |
| Success reward | 1.0 | 10.0 | 10x higher ✓ |
| Offline demos | 15 | 20-30 | Need more? |

## Config Changes

```json
{
    "temperature_init": 0.01,
    "utd_ratio": 20
}
```

## Key Insights from HIL-SERL Paper

1. **Sample efficiency**: HIL-SERL achieves 100% success in 1-2.5 hours of training
2. **Intervention strategy**: Allow exploration early, then intervene to correct off-track behavior
3. **Intervention rate should drop**: If intervention stays high, hyperparameters are wrong
4. **Ego-centric framing**: Proprioception expressed relative to initial EE frame for generalization
5. **Equal buffer sampling**: 50% from demo buffer, 50% from online buffer

## Action Items

1. ✅ Fix reward structure (+10 success, -0.05 step penalty)
2. ✅ Fix utd_ratio: 2 → 20
3. ✅ Fix temperature_init: 0.2 → 0.01
4. ⬜ Consider collecting more demos (current: 15, recommended: 20-30)
5. ⬜ Clear buffer and restart training with new hyperparameters

## References

- [HIL-SERL Paper](https://hil-serl.github.io/)
- [LeRobot HIL-SERL Docs](https://huggingface.co/docs/lerobot/en/hilserl)
- [RLPD Paper](https://arxiv.org/abs/2302.02948)
- [SERL Paper](https://arxiv.org/abs/2401.16013)
