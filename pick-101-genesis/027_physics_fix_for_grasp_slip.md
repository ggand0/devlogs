# Devlog 027: Physics Fix for Grasp Slip Issue

**Date:** 2026-01-14

## Problem

v19 training achieved only 20% success rate at 1M steps. Eval videos showed the agent attempting to grasp and lift the cube, but it frequently slipped out of the gripper. IK grasp test achieved 100% success, confirming physics works but RL exploration struggles.

## Root Cause Analysis

Three physics parameters made grasping unreliable during RL exploration:

1. **DR friction minimum too low** - effective friction at cube=0.3 was only 0.77
2. **Gripper kp weaker than arm** - 500 vs 1000, insufficient grip force
3. **Only 2 physics substeps** - contact resolution potentially unstable

## Friction Analysis

**Gripper material:** PLA (3D printed plastic)
**Cube material:** Wood
**Realistic PLA-on-wood friction:** ~0.4-0.5

MuJoCo/Genesis calculates effective friction as geometric mean of both surfaces:
```
effective_friction = sqrt(finger_friction × cube_friction)
```

Current MJCF uses inflated finger friction (2.0) to compensate for PLA's low real friction.

| Cube Friction | Finger (MJCF) | Effective Friction |
|---------------|---------------|-------------------|
| 0.3 (old min) | 2.0           | **0.77** (too slippery) |
| 0.5 (new min) | 2.0           | **1.0** |
| 1.0 (default) | 2.0           | 1.41 |
| 1.5 (max)     | 2.0           | 1.73 |

## Changes Made

### 1. Increase DR friction minimum: 0.3 → 0.5
**File:** `src/envs/lift_cube_env.py:66`
```python
# Before
dr_friction_range: tuple[float, float] = (0.3, 1.5),

# After
dr_friction_range: tuple[float, float] = (0.5, 1.5),
```
Raises effective friction from 0.77 → 1.0 at minimum DR setting.

### 2. Increase gripper kp: 500 → 1000
**File:** `src/envs/lift_cube_env.py:386`
```python
# Before
kp[self.gripper_dof_idx] = 500.0

# After
kp[self.gripper_dof_idx] = 1000.0
```
Matches arm joint strength for firmer grip.

### 3. Increase physics substeps: 2 → 4
**File:** `src/envs/lift_cube_env.py:149`
```python
# Before
substeps=2,

# After
substeps=4,
```
More stable contact resolution during gripper closure and lift.

### 4. Increase eval frequency: 100k → 50k
**File:** `src/training/cfgs/genesis_lift.yaml:41`
```yaml
eval_every_steps: 50000
```
More frequent checkpoints to catch improvements earlier.

## Verification

IK grasp test with new physics: **100% success (10/10)**

```
$ uv run python scripts/ik_grasp_multipos.py
...
Success rate: 100% (10/10)
```

## Training Run

Started 1M step training with physics fixes:
```bash
uv run python src/training/train_drqv2.py
```

Run directory: `runs/drqv2/20260114_HHMMSS/`

## Expected Outcome

- Training success should be > 0% (was always 0% before)
- Eval success should improve from baseline 20%
- Less cube slipping in eval videos

## References

- [Devlog 026](026_v19_training_results_analysis.md) - Previous training results
- [Devlog 016](016_mujoco_menagerie_collision_fix.md) - Original collision fix
- [PLA friction research](https://journals.sagepub.com/doi/abs/10.1177/1350650120966407)
