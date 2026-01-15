# Devlog 009: IK Controller and PD Gains Fix

## Problem

1. **Arm twitching** in rendered videos during training
2. **Cube bouncing** on initial spawn
3. **Training env not matching** the smooth motion from test_splitview_pick.py

## Root Causes

### 1. Wrong IK Controller
The training env was using `SimpleIKController` which had different behavior than the `JacobianIKController` used in test_splitview_pick.py:
- No `locked_joints` support for wrist orientation
- Different gain handling (default 1.0 vs 0.5)
- Used `ee_link` (gripper body) instead of `tcp_link` (tool center point)

### 2. Wrong PD Gains
Genesis env was using arbitrary gains:
```python
# Before (wrong)
kp[arm] = 500.0
kv[arm] = 50.0  # Way too high!
kp[gripper] = 100.0
kv[gripper] = 10.0
```

MuJoCo so101_new_calib.xml specifies actual STS3215 servo gains:
```xml
<position kp="998.22" kv="2.731" forcerange="-2.94 2.94"/>
```

The kv=50 was ~18x too high, causing instability.

### 3. No Cube Settling
The cube was spawned and immediately used without letting physics settle, causing bounce.

## Fixes Applied

### 1. Replaced IK Controller
Replaced `SimpleIKController` and `GenesisIKController` with `JacobianIKController` from test_splitview_pick.py:

```python
# src/controllers/ik_controller.py
class JacobianIKController:
    def __init__(self, robot, tcp_link, device="cuda", damping=0.1, max_dq=0.5):
        self.tcp_link = tcp_link  # Use TCP, not gripper body
        self.arm_dof_indices = [6, 7, 8, 9, 10]
        ...

    def step_toward_target(self, target_pos, gripper_val, gain=0.5, locked_joints=None):
        # Supports locked_joints for wrist orientation
        # Uses gain=0.5 by default
        ...
```

### 2. Fixed PD Gains
Updated to match MuJoCo STS3215 servo gains:

```python
# src/envs/lift_cube_env.py
# Arm joints (STS3215 servo gains from MuJoCo)
for i in range(self.arm_dof_start, self.arm_dof_end):
    kp[i] = 998.0
    kv[i] = 2.7
# Gripper (same servo)
kp[self.gripper_dof_idx] = 998.0
kv[self.gripper_dof_idx] = 2.7
```

### 3. Added Cube Settling
Added velocity zeroing and settling steps after reset:

```python
def reset(self, env_ids=None):
    self.scene.reset(envs_idx=...)

    # Zero cube velocity to prevent bouncing
    if self.cube_link is not None:
        dofs_vel = self.robot.get_dofs_velocity()
        dofs_vel[:, :6] = 0
        self.robot.set_dofs_velocity(dofs_vel)

    # Let cube settle (100 physics steps)
    for _ in range(100):
        self.scene.step()
```

### 4. Added Pre-grasp Motion in Reset
For curriculum_stage >= 3, the reset now performs actual IK motion to position the gripper above the cube:

```python
def _reset_gripper_near_cube(self, env_ids):
    cube_pos = self.cube_link.get_pos()
    above_pos = cube_pos.clone()
    above_pos[:, 0] += -0.015  # finger_width_offset
    above_pos[:, 2] += 0.035   # grasp_z_offset + height_offset

    # Move to above position using IK (100 steps)
    for _ in range(100):
        joint_targets = self.ik_controller.compute_joint_targets(above_pos, gripper_open)
        joint_targets[:, 9] = torch.pi / 2   # wrist_flex locked
        joint_targets[:, 10] = torch.pi / 2  # wrist_roll locked
        self.robot.control_dofs_position(joint_targets)
        self.scene.step()
```

## Files Modified

- `src/controllers/ik_controller.py` - Replaced with JacobianIKController
- `src/controllers/__init__.py` - Updated exports
- `src/envs/lift_cube_env.py` - PD gains, reset sequence, IK controller usage
- `tests/verify_training_env.py` - Verification script

## Verification

Video at `tests/env_reset_check.mp4` shows:
- Cube stable on table (no bouncing)
- Arm positioned above cube after reset
- Smooth motion during steps

## Notes

- Cube settles at z~0.027 instead of z=0.015 due to table collision height
- The `tcp_link` is the tool center point (between fingers), not the gripper body
- Wrist joints (indices 3, 4 in arm, or 9, 10 in full qpos) should be locked at pi/2 for top-down grasp
