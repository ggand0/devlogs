# Environment Implementation Progress

## Summary

Successfully implemented basic Genesis environment for SO-101 lift cube task.

## What Works

### MJCF Loading
- Genesis loads `lift_cube.xml` successfully
- All robot links detected: world, cube, base, shoulder, upper_arm, lower_arm, wrist, gripper, moving_jaw_so101_v1
- 12 DOFs total: 6 (cube free joint) + 5 (arm) + 1 (gripper)

### Observation Space (21 dims)
- Joint positions (6): arm joints + gripper
- Joint velocities (6)
- Gripper position (3): from "gripper" link
- Gripper orientation (3): quaternion to euler conversion
- Cube position (3): from "cube" link

### Action Space (4 dims)
- Delta X, Y, Z for end-effector (scaled by 0.02m/step)
- Gripper open/close (-1 to 1)

### Reward
- Using ported v11 reward function from pick-101
- Works correctly with tensor-to-numpy conversion

### Performance
- ~250-400 FPS per environment on Radeon RX 7900 XTX
- Vulkan backend working correctly

## Completed

### IK Controller
Implemented proper IK controllers using Genesis APIs:

1. **GenesisIKController** (default): Uses Genesis's built-in `inverse_kinematics()` method
   - Damped least-squares IK solver
   - Configurable parameters: damping, max_solver_iters, pos_tol
   - Only solves for arm DOFs (6-10), keeping cube DOFs (0-5) untouched

2. **SimpleIKController** (fallback): Jacobian-based IK
   - Uses `robot.get_jacobian()` to compute Jacobian matrix
   - Implements damped least-squares: dq = (J^T J + λ²I)^-1 J^T * error
   - Falls back to pseudoinverse if linear solve fails

Usage: `LiftCubeEnv(use_builtin_ik=True)` for Genesis IK, `False` for Jacobian-based.

## Known Issues

### Contact Detection (Simplified)
Current implementation just checks if any contacts exist:
```python
contacts = self.robot.get_contacts()
has_static = valid.any(dim=-1) if valid.dim() > 1 else valid
has_moving = has_static  # Simplified
```

**Needs:**
- Filter contacts by specific link/geom (cube vs finger pads)
- Use `get_contacts(with_entity=...)` to get cube-specific contacts

### MuJoCo Sites
Genesis doesn't expose MJCF sites as links. The "gripperframe" site from MuJoCo isn't accessible directly.

**Workaround:** Using "gripper" body link instead. Must compute TCP offset manually.

### IK DOF Indexing Mismatch
Genesis IK uses 13 DOFs internally (cube quaternion = 7 DOFs), while robot qpos uses 12 DOFs (cube euler = 6 DOFs).

**Mapping:**
- IK output indices 7-11 → Robot qpos indices 6-10 (arm joints)
- IK output index 12 → Robot qpos index 11 (gripper)

When using `dofs_idx_local` in `inverse_kinematics()`, indices refer to the 13-DOF IK space, not the 12-DOF qpos space.

## Top-Down Pick Test Progress

### Test File
`tests/test_topdown_pick.py` - Porting from `pick-101/tests/test_topdown_pick.py`

### Sequence (matching MuJoCo implementation)
1. Move above block (30mm height), gripper OPEN
2. Move down to block, gripper OPEN
3. Close gripper gradually
4. Lift up
5. Hold

### Key Differences from MuJoCo

| Aspect | MuJoCo | Genesis |
|--------|--------|---------|
| IK target | gripperframe site (TCP) | gripper link body |
| IK locked joints | `locked_joints=[3,4]` parameter | Exclude wrist DOFs from `dofs_idx_local` |
| TCP offset | Direct site position | Manual calculation: `pos + rotate(offset, quat)` |
| DOF representation | 12 DOFs (euler) | 13 DOFs in IK (quaternion), 12 in qpos |

### TCP Offset Calculation
With top-down pose (wrist_flex=π/2, wrist_roll=π/2):
- gripperframe site in MJCF: `pos="0.0 0.0 -0.0981274"`
- TCP is ~9.8cm below gripper link in world Z
- To place TCP at height Z, target gripper link at Z + 0.098

### Current Issues (2025-01-10)

**Problem:** IK target not being reached
- Target gripper link: [0.25, -0.015, 0.118]
- Actual gripper link: [0.159, 0.0, 0.101]
- Error: ~9cm

**Possible causes:**
1. IK solution not being applied correctly
2. Position control PD gains insufficient
3. Joint limits preventing motion
4. IK solver not converging for some targets

**Verified working:**
- IK solver runs without errors
- Robot control_dofs_position() accepts targets
- Gripper open/close works (0.035 → 0.004)

### Isolated IK Test Result
When testing IK in isolation (no grasp sequence):
```
Target link: [0.25, 0.0, 0.115]
Final link:  [0.241, 0.0, 0.117]
Error: 0.0094 (9.4mm)
```
This shows IK CAN reach targets when properly configured. Issue is in the full test sequence.

## Next Steps

1. **Debug IK application** in top-down pick test
2. **Verify joint limits** for arm DOFs
3. **Check PD gain values** for position control convergence
4. **Implement proper grasp detection** using contact filtering
5. **Training Loop**: Set up PPO training after grasp works

## Code Structure

```
src/
├── envs/
│   ├── __init__.py           # Exports LiftCubeEnv, REWARD_FUNCTIONS
│   ├── lift_cube_env.py      # Main environment (500+ lines)
│   └── rewards/
│       ├── __init__.py       # Exports v11, v19 reward functions
│       └── lift_rewards.py   # Ported from pick-101
└── controllers/
    ├── __init__.py           # Exports GenesisIKController, SimpleIKController
    └── ik_controller.py      # IK controllers for arm manipulation
```

## Dependencies

- `genesis-world>=0.3.0`
- `numpy>=2.0.0` (required for Genesis pyrender compatibility)
- `torch>=2.8.0` (ROCm build)

## Test Command

```bash
cd /home/gota/ggando/ml/pick-101-genesis
uv run python -c "
from src.envs import LiftCubeEnv
import torch

env = LiftCubeEnv(show_viewer=False, num_envs=1)
obs, info = env.reset()
for i in range(10):
    action = torch.zeros(1, 4, device=env.device)
    obs, reward, term, trunc, info = env.step(action)
    print(f'Step {i}: reward={reward.item():.3f}')
"
```
