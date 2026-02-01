# Gripper Action Format Mismatch in HIL-SERL

## Problem

Gripper works in episode 1 but fails to reopen in episode 2+. User did NOT intervene in episode 1.

**Symptoms:**
- Reset opens gripper correctly (50.0) at start of each episode
- During episode, gripper closes and won't reopen
- Policy appears to command "open" but gripper doesn't respond

## Root Cause Analysis

### Action Space Mismatch

**The action space** in `gym_manipulator.py` defined gripper as [0, 2]:
```python
if self.use_gripper:
    action_dim += 1
    bounds["min"] = np.concatenate([bounds["min"], [0]])   # 0
    bounds["max"] = np.concatenate([bounds["max"], [2]])   # 2
```

**SAC policy** outputs [-1, 1] due to tanh squashing.

### What Happens

- **Episode 1**: `action_space.sample()` returns gripper in [0, 2] → works
- **Episode 1 actions** stored in replay buffer with gripper in [0, 2]
- **Policy learns** from [0, 2] targets but can only output [-1, 1] (tanh)
- **Episode 2+**: Policy outputs [-1, 1] but learned wrong mapping from [0, 2] data

### The Mismatch

Policy learns:
- Target 0.0 (close) → Policy outputs ~0.0
- Target 2.0 (open) → Policy outputs 1.0 (saturated tanh, can't reach 2.0)
- Target 1.0 (no-op) → Policy outputs ~0.5-1.0

Policy outputs [0, 1] for gripper but robot expects [-1, 1]:
- ~0.0 = close (policy semantics) but robot interprets as no-op
- ~1.0 = open (both agree)

### Robot Code Interpretation

Original code in `so101_follower_end_effector.py`:
```python
gripper_delta = (action[-1] - 1) * self.config.max_gripper_pos
```

With SAC output 1.0 (trying to open):
- delta = (1 - 1) * 50 = 0 → NO CHANGE (bug!)

With SAC output 0.0 (trying to close):
- delta = (0 - 1) * 50 = -50 → closes correctly

**This explains why gripper closes but can't reopen.**

### Online Learning Format Issue

Teleoperation code in `gym_manipulator.py` was storing gripper actions in [0, 1, 2] format:
```python
if normalized_delta >= 0.3:
    gripper_action = 2   # open
elif normalized_delta <= 0.1:
    gripper_action = 0   # close
else:
    gripper_action = 1   # no-op
```

This perpetuates the format mismatch during online learning.

## Fixes Applied

### 1. Action space bounds (`gym_manipulator.py` line 503-506)

Changed gripper action space from [0, 2] to [-1, 1]:
```python
if self.use_gripper:
    action_dim += 1
    # Gripper action space [-1, 1] to match SAC tanh output
    bounds["min"] = np.concatenate([bounds["min"], [-1]])  # was [0]
    bounds["max"] = np.concatenate([bounds["max"], [1]])   # was [2]
```

### 2. Robot gripper interpretation (`so101_follower_end_effector.py`)

Changed to direct scaling:
```python
# Handle gripper: [-1, 1] where -1=close, 0=no-op, 1=open
gripper_action = action[-1]

# Convert legacy [0, 2] format if detected
if gripper_action > 1.0:
    gripper_action = gripper_action - 1.0

gripper_delta = gripper_action * self.config.max_gripper_pos
```

### 3. Teleoperation action storage (`gym_manipulator.py` line 2226-2233)

Changed from [0, 1, 2] to [-1, 0, 1]:
```python
if normalized_delta >= 0.3:
    gripper_action = 1   # open (was 2)
elif normalized_delta <= -0.3:
    gripper_action = -1  # close (was 0)
else:
    gripper_action = 0   # no-op (was 1)
```

## Summary

The root cause was the action space defining gripper bounds as [0, 2] while SAC outputs [-1, 1]. This caused:
1. Random sampling in episode 1 to produce [0, 2] gripper actions
2. Replay buffer to store [0, 2] gripper actions
3. Policy to learn wrong mapping (can't output 2.0 with tanh)
4. Episode 2+ policy outputs don't match learned targets

## Files Modified

| File | Change |
|------|--------|
| `lerobot/scripts/rl/gym_manipulator.py:503-506` | Action space gripper bounds [0,2] → [-1,1] |
| `lerobot/scripts/rl/gym_manipulator.py:2226-2233` | Teleop action format [0,1,2] → [-1,0,1] |
| `lerobot/robots/so101_follower/so101_follower_end_effector.py` | Gripper delta uses direct scaling |

## Note

Must clear replay buffer / start fresh training after this fix. Old data has gripper in [0, 2] format.

## Research Findings

### SERL/HIL-SERL Gripper Handling

From [SERL GitHub](https://github.com/rail-berkeley/serl) and [HIL-SERL paper](https://hil-serl.github.io/):
- Original SERL uses **discrete** gripper actions (open/close binary)
- A separate **DQN grasp critic** is trained for gripper decisions
- Gripper actions are treated differently from continuous arm control
- Small penalty for gripper action changes recommended for training speed

### LeRobot HIL-SERL Implementation

From [LeRobot HIL-SERL docs](https://huggingface.co/docs/lerobot/en/hilserl):
- Uses `GripperVelocityToJoint` processor step for gripper control
- Gripper config includes `use_gripper` and `gripper_penalty` settings
- Actions are in end-effector space: `[delta_x, delta_y, delta_z, gripper]`

### SAC and Action Space Mismatch

From [Stable Baselines3 SAC docs](https://stable-baselines3.readthedocs.io/en/master/modules/sac.html) and [research](https://arxiv.org/html/2410.16739v1):
- SAC uses tanh squashing to bound actions to [-1, 1]
- **Critical**: "When using offline data with SAC, scaling/unscaling between the environment's action space and the [-1,1] range is required"
- The tanh transformation introduces non-linear distortion
- Mismatch between sampled actions and most probable actions can occur

### Solutions

1. **Transform offline data** to [-1, 1] before BC pretraining
2. **Use discrete gripper** with separate DQN critic (original SERL approach)
3. **Action space normalization** in data loading pipeline

## Sources

- [SERL GitHub](https://github.com/rail-berkeley/serl)
- [HIL-SERL Paper](https://hil-serl.github.io/static/hil-serl-paper.pdf)
- [LeRobot HIL-SERL Docs](https://huggingface.co/docs/lerobot/en/hilserl)
- [Stable Baselines3 SAC](https://stable-baselines3.readthedocs.io/en/master/modules/sac.html)
- [Corrected SAC Paper](https://arxiv.org/html/2410.16739v1)
