# 010: Arm Motion Fix and MuJoCo Pre-Grasp Matching

## Problem

The robot arm was exhibiting several issues:
1. Arm dropping due to gravity at episode start
2. Erratic/fast motion during grasping
3. Pre-grasp motion not matching the verified MuJoCo implementation

## Root Cause Analysis

### Gravity Drop
The Genesis environment was resetting without initializing the arm to a stable pose. Unlike MuJoCo which has implicit damping, Genesis requires explicit arm positioning during reset to prevent gravity-induced motion.

### Erratic Motion
The `_compute_ik()` function was missing critical parameters:
- No `locked_joints=[3,4]` to keep wrist orientation fixed for top-down grasp
- Using default `gain=0.5` instead of smoother `gain=0.3`

### PD Gains Verification
Investigated whether `kp=1000, kv=50` was correct for SO-101:
- MuJoCo uses `kp=998.22`, so `kp=1000` is correct
- MuJoCo uses `kv=2.731` but Genesis needs higher damping
- `kv=50` provides overdamped response (damping ratio ζ ≈ 4) for stable motion in Genesis

## Solution

### 1. Arm Initialization in reset()
```python
# Initialize arm to home pose
init_target = torch.zeros(self.num_envs, 12, device=self.device)
init_target[:, 9] = torch.pi / 2   # wrist_flex
init_target[:, 10] = torch.pi / 2  # wrist_roll
init_target[:, 11] = 1.0           # gripper open

# Let arm reach home pose (100 physics steps)
for _ in range(100):
    self.robot.control_dofs_position(init_target)
    self.scene.step()
```

### 2. Fixed _compute_ik()
```python
def _compute_ik(self, target_pos, gripper_action):
    joint_targets = self.ik_controller.compute_joint_targets(
        target_pos, gripper_action, gain=0.3, locked_joints=[3, 4]
    )
    # Ensure wrist joints stay at pi/2
    joint_targets[:, 9] = torch.pi / 2
    joint_targets[:, 10] = torch.pi / 2
    return joint_targets
```

### 3. Verification Script
Created `tests/verify_training_env.py` that visualizes the reset motion matching MuJoCo's `_reset_with_cube_in_gripper()`:

| Phase | Steps | Description |
|-------|-------|-------------|
| 1 | 100 | Arm initialization (wrist at pi/2) |
| 2 | 50 | Cube settling |
| 3 | 300 | Move above cube |
| 4 | 200 | Move down to grasp height |

## MuJoCo Reference

The pre-grasp motion was matched against `/home/gota/ggando/ml/pick-101/src/envs/lift_cube.py`:

```python
# MuJoCo _reset_with_cube_in_gripper() lines 357-373
# Step 1: Move above block (300 steps)
for _ in range(300):
    ctrl = self.ik.step_toward_target(above_pos, gripper_action=gripper_open,
                                       gain=0.5, locked_joints=locked_joints)

# Step 2: Move down to block (200 steps)
for _ in range(200):
    ctrl = self.ik.step_toward_target(grasp_target, gripper_action=gripper_open,
                                       gain=0.5, locked_joints=locked_joints)
```

## Key Parameters

| Parameter | Genesis Value | MuJoCo Value | Notes |
|-----------|---------------|--------------|-------|
| kp (arm) | 1000.0 | 998.22 | Stiffness |
| kv (arm) | 50.0 | 2.731 | Genesis needs higher damping |
| kp (gripper) | 500.0 | - | Lower stiffness for gripper |
| IK gain | 0.3 | 0.5 | Smoother motion |
| locked_joints | [3, 4] | [3, 4] | Wrist flex/roll at pi/2 |
| gripper_open | 0.5 | 0.3 (action) | Half-open DOF position |

## Verification

Run the verification script:
```bash
uv run python tests/verify_training_env.py
```

Output video: `tests/env_reset_check.mp4` (160 frames)

## Files Changed

- `src/envs/lift_cube_env.py`: Added arm initialization in reset(), fixed _compute_ik()
- `src/controllers/ik_controller.py`: Cleaned up JacobianIKController
- `tests/verify_training_env.py`: New verification script for reset motion
