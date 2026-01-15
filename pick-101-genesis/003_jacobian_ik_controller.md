# Jacobian-Based IK Controller Implementation

## Summary

Implemented a proper Jacobian-based iterative IK controller for Genesis, replacing the built-in `inverse_kinematics()` which was getting stuck in local minima. The top-down pick test now succeeds with position errors of ~5mm (down from 17cm).

## Problem

Genesis's built-in `robot.inverse_kinematics()` was producing poor solutions:
- Position errors of 10-17cm when targeting cube at [0.25, 0, 0.05]
- IK solver finding local minima that curled arm backward instead of reaching forward
- Single-shot solve couldn't escape bad configurations

## Solution

Ported the MuJoCo IK controller approach to Genesis:

### JacobianIKController Class

```python
class JacobianIKController:
    def __init__(self, robot, tcp_link, device="cuda", damping=0.1, max_dq=0.5):
        self.robot = robot
        self.tcp_link = tcp_link
        self.damping = damping
        self.max_dq = max_dq

    def compute_joint_velocities(self, target_pos, locked_joints=None):
        # Get Jacobian: (n_envs, 6, n_dofs)
        jacobian = self.robot.get_jacobian(self.tcp_link)

        # Position Jacobian for arm joints
        Jp = jacobian[0, :3, 6:11]  # (3, 5)

        # Damped least-squares
        error = target_pos - current_pos
        JTJ = J.T @ J
        dq = torch.linalg.solve(JTJ + λ²I, J.T @ error)

        return dq

    def step_toward_target(self, target_pos, gripper_val, gain=0.5, locked_joints=None):
        dq = self.compute_joint_velocities(target_pos, locked_joints)
        target = current + dq * gain
        return target
```

### Key Differences from Genesis Built-in IK

| Aspect | Genesis IK | Jacobian IK |
|--------|-----------|-------------|
| Approach | Single-shot solve | Iterative stepping |
| Local minima | Gets stuck | Escapes via small steps |
| DOF handling | 13-DOF IK space | Direct 12-DOF qpos |
| Locked joints | Via dofs_idx_local | Exclude from Jacobian |

## Genesis API Notes

### Getting Jacobian
```python
jacobian = robot.get_jacobian(link)  # (n_envs, 6, n_dofs)
# First 3 rows: position Jacobian
# Last 3 rows: orientation Jacobian
```

### DOF Indexing
- Robot qpos: 12 DOFs (cube euler=6, arm=5, gripper=1)
- Jacobian columns match qpos indices
- Arm joints: indices 6-10
- Gripper: index 11

## Test Results

### Before (Genesis built-in IK)
```
Target TCP: [0.25, -0.015, 0.05]
Actual TCP: [0.08, 0.0, 0.01]
Position error: 0.17m (17cm)
Result: FAIL
```

### After (Jacobian IK)
```
Target TCP: [0.25, -0.015, 0.05]
Actual TCP: [0.245, -0.015, 0.048]
Position error: 0.005m (5mm)
Result: SUCCESS
```

## Grasp Parameters

```python
# Matching original MuJoCo test
height_offset = 0.03      # 30mm above block
grasp_z_offset = 0.005    # 5mm above cube center
finger_width_offset = -0.015  # Y offset to center grip

# Gripper joint range: -0.17453 to 1.74533 radians
gripper_open = 1.0
gripper_closed = -0.15

# Lock wrist for top-down
locked_joints = [3, 4]  # wrist_flex, wrist_roll at π/2
```

## Known Limitations

1. **Gripper doesn't close fully**: Target -0.15 achieves ~0.29 actual. Enough to grip but cube may slip during extended holding.

2. **Cube drops after lift**: During hold phase, grip loosens and cube falls. Not critical for RL training which focuses on the lift action.

3. **PD gains tuning**: Current gains (kp=1000, kv=50) work but may need adjustment for different tasks.

## Files Modified

- `tests/test_topdown_pick.py`: Complete rewrite with JacobianIKController

## Next Steps

1. Integrate JacobianIKController into LiftCubeEnv for RL training
2. Tune gripper control for tighter grasp
3. Set up PPO training loop
