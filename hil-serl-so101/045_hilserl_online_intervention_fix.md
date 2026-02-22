# Devlog 045: HIL-SERL Online Intervention Zero-Action Bug Fix

**Date**: 2025-01-18
**Status**: Complete

## Problem

During HIL-SERL online training, human interventions were being stored with **zero actions** in the online buffer. This is the same root cause as the offline dataset issue (devlog 044), but affects the online training path.

When intervening during training:
- Joint mirroring worked correctly (robot followed leader)
- But the action stored in the online buffer was `[0.0, 0.0, 0.0]`
- The policy learned that zero-actions are correct responses during human guidance

## Root Cause Analysis

### The Bug in BaseLeaderControlWrapper

In `gym_manipulator.py`, `BaseLeaderControlWrapper.__init__` only initialized kinematics for URDF-based robots:

```python
# Line 1642-1647 (before fix)
self.kinematics = None
if hasattr(env.unwrapped.robot.config, 'urdf_path') and hasattr(env.unwrapped.robot.config, 'target_frame_name'):
    self.kinematics = RobotKinematics(
        urdf_path=env.unwrapped.robot.config.urdf_path,
        target_frame_name=env.unwrapped.robot.config.target_frame_name,
    )
```

For SO-101 with `so101_follower_end_effector` robot type (which uses MuJoCo, not URDF), `self.kinematics` remained `None`.

### The Zero-Action Fallback

When `kinematics is None`, `_handle_intervention()` fell back to returning dummy zeros:

```python
# Line 1816-1822 (before fix)
else:
    # Fallback: when no kinematics available, store leader positions for later use
    self.unwrapped._leader_positions = {name: leader_pos_dict[name] for name in leader_pos_dict}
    # Return dummy end-effector action to satisfy wrapper interface
    action = np.array([0.0, 0.0, 0.0])
```

### Data Flow During Online Training

1. Human intervenes → `_handle_intervention()` called
2. Joint mirroring works via `_leader_positions`
3. But action returned is `[0.0, 0.0, 0.0]`
4. Actor stores this zero action in `Transition.action`
5. Online buffer contains zero intervention actions
6. Learner trains on corrupted demonstrations

## Fix Applied

Added MuJoCo FK support to `BaseLeaderControlWrapper`, mirroring the pattern used in `FullProprioceptionWrapper`.

### Fix 1: Initialize MuJoCo FK in __init__

```python
# Initialize robot control - support both URDF and MuJoCo-based robots
self.kinematics = None
self.use_mujoco_fk = False

if hasattr(env.unwrapped.robot.config, 'urdf_path') and hasattr(env.unwrapped.robot.config, 'target_frame_name'):
    self.kinematics = RobotKinematics(
        urdf_path=env.unwrapped.robot.config.urdf_path,
        target_frame_name=env.unwrapped.robot.config.target_frame_name,
    )
elif hasattr(env.unwrapped.robot, 'mj_data'):
    # MuJoCo-based robot (like SO101FollowerEndEffector)
    self.use_mujoco_fk = True
```

### Fix 2: Use MuJoCo FK in _handle_intervention()

Added a new code path for MuJoCo-based FK that:
1. Computes proper EE delta actions via MuJoCo FK
2. Preserves direct joint mirroring for responsive teleoperation
3. Saves/restores MuJoCo state to avoid corrupting robot internals

```python
elif self.use_mujoco_fk:
    # MuJoCo-based FK for SO101FollowerEndEffector and similar robots
    import mujoco
    robot = self.unwrapped.robot
    mj_model = robot.mj_model
    mj_data = robot.mj_data
    ee_site_id = robot.ee_site_id

    # Number of DOF (exclude gripper for FK)
    num_dof = min(len(leader_pos) - 1, mj_model.nq) if len(leader_pos) > 1 else mj_model.nq

    # Save original qpos to restore after FK computation
    original_qpos = mj_data.qpos.copy()

    # Compute leader EE position via FK
    leader_joints_rad = np.deg2rad(leader_pos[:num_dof])
    mj_data.qpos[:num_dof] = leader_joints_rad
    mujoco.mj_forward(mj_model, mj_data)
    leader_ee = mj_data.site_xpos[ee_site_id].copy()

    # Compute follower EE position via FK
    follower_joints_rad = np.deg2rad(follower_pos[:num_dof])
    mj_data.qpos[:num_dof] = follower_joints_rad
    mujoco.mj_forward(mj_model, mj_data)
    follower_ee = mj_data.site_xpos[ee_site_id].copy()

    # Restore original qpos so robot's internal state isn't corrupted
    mj_data.qpos[:] = original_qpos
    mujoco.mj_forward(mj_model, mj_data)

    # Store leader positions for direct joint mirroring (for responsive teleoperation)
    self.unwrapped._leader_positions = {name: leader_pos_dict[name] for name in leader_pos_dict}

    # Compute delta and normalize for the EE action (stored in buffer for learning)
    action = np.clip(leader_ee - follower_ee, -self.end_effector_step_sizes, self.end_effector_step_sizes)
    action = action / self.end_effector_step_sizes
```

**Key insight**: The fix must do BOTH:
- **Direct joint mirroring** (`_leader_positions`) for responsive real-time teleoperation
- **EE delta action computation** for the buffer so the policy can learn from demonstrations

## Why This Matters

| Training Mode | Before Fix | After Fix |
|---------------|-----------|-----------|
| Offline buffer | Zero actions stored | Zero action detection + FK conversion (devlog 044) |
| Online buffer | Zero actions stored during interventions | MuJoCo FK computes proper EE delta actions |

Without this fix, all human interventions during online training would be stored as "do nothing" commands, teaching the policy the opposite of what the human demonstrated.

## Files Modified

- `lerobot/src/lerobot/scripts/rl/gym_manipulator.py`
  - `BaseLeaderControlWrapper.__init__`: Added MuJoCo FK detection
  - `_handle_intervention()`: Added MuJoCo FK code path

## Verification

To verify the fix is working, check the actor logs during intervention:
1. Ensure `BaseLeaderControlWrapper` detects MuJoCo FK: `self.use_mujoco_fk = True`
2. Verify intervention actions are non-zero: `info["action_intervention"]` should have meaningful values

## Related

- Devlog 044: HIL-SERL Training Fixes (offline buffer zero actions)
- Devlog 042: DrQ-v2 Encoder Synchronization Fix
- Devlog 043: Teleoperation Unit Mismatch Fix
