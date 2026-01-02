# Cartesian Action Space with IK Controller

## Motivation

Joint-space control (6D action space of joint velocities) proved difficult for RL exploration. Random joint actions produce chaotic arm movements that rarely contact the cube. The HER experiment failed with 0% success because the agent never accidentally touched the cube.

Solution: Use an IK (Inverse Kinematics) controller internally, letting the agent control the end-effector in Cartesian space (delta XYZ + gripper).

## Implementation

### IK Controller (`controllers/ik_controller.py`)

Damped least-squares IK using MuJoCo's Jacobian:

```python
# Compute Jacobian at end-effector site
mujoco.mj_jacSite(model, data, jacp, jacr, ee_site_id)
J = jacp[:, :n_arm_joints]  # Only arm joints, not gripper

# Damped least-squares: (J^T J + λ²I)^-1 J^T × error
JTJ = J.T @ J
damping_matrix = damping**2 * np.eye(n_arm_joints)
dq = np.linalg.solve(JTJ + damping_matrix, J.T @ pos_error)
```

This converts Cartesian position targets to joint velocities while handling singularities gracefully.

### Cartesian Environment (`envs/lift_cube_cartesian.py`)

- **Action space**: 4D `[-1, 1]` → delta XYZ (scaled by 2cm/step) + gripper open/close
- **Observation**: 21D (joint pos/vel + gripper pos/euler + cube pos)
- **Reward**: Robosuite-style dense reward
  - Reach: `1.0 - tanh(10 * distance)` (smooth gradient)
  - Grasp: +0.25 if cube pinched between both gripper parts
  - Lift: +1.0 if cube_z > 0.02m
  - Success: +10.0 if lifted to target height and held

### Grasp Detection

Requires contact on BOTH gripper parts (robosuite-style):
- Static gripper body (geoms 25-28)
- Moving jaw (geoms 29-30)

This prevents reward hacking by just pushing with closed gripper.

## Training Results (500k steps)

```
Config: configs/lift_cartesian_500k.yaml
Training time: ~1h 53min
Final eval reward: 225.17 +/- 0.34
Success rate: 0%
```

### Behavior Analysis

The agent learned to:
1. Approach the cube quickly (reach reward maximized)
2. Close gripper around cube (grasp detected: contacts=(True, True))
3. Maintain contact position (dist ~0.007m)

**But failed to lift because:**
- Gripper contacts top surface + one side, not opposite sides
- Cube pushed down slightly (z: 0.01 → 0.007)
- No reward gradient for attempting to lift

Eval output:
```
step=50: gripper=-0.168, dist=0.007, cube_z=0.007, grasp=True, contacts=(True, True)
step=100: gripper=-0.168, dist=0.007, cube_z=0.007, grasp=True, contacts=(True, True)
Episode: reward=225.36, success=False, cube_z=0.007
```

### Reward Analysis

Per-step breakdown when grasping:
- Reach reward: ~1.0 (very close to cube)
- Grasp bonus: +0.25
- Lift bonus: 0 (cube_z=0.007 < 0.02 threshold)
- **Total: ~1.25/step × 200 steps ≈ 250**

The lift bonus threshold (0.02m) was never achievable because:
1. Cube starts at z=0.01
2. Gripper pressing from above pushes it down
3. No gradient to encourage lifting attempts

## Comparison with Joint-Space Training

| Metric | Joint-Space (LiftCubeEnv) | Cartesian (LiftCubeCartesianEnv) |
|--------|---------------------------|----------------------------------|
| Action dims | 6 (joint velocities) | 4 (delta XYZ + gripper) |
| 500k reward | ~61.5 | ~225 |
| Learned reach | Partial | Yes |
| Learned grasp | No | Yes |
| Contacts | Rare | Consistent |
| Success rate | 0% | 0% |

Cartesian space dramatically improved learning - agent consistently reaches and grasps the cube. The failure is now at the lift stage, not exploration.

## Key Insights

1. **IK abstracts kinematics**: Random Cartesian actions naturally explore the workspace
2. **Robosuite-style grasp detection works**: Prevents pushing-with-closed-gripper hack
3. **Binary lift reward insufficient**: Need continuous reward for lift height
4. **Approach angle matters**: Top-down grasp doesn't create liftable pinch

## Next Steps

1. **Add continuous lift reward**: `reward += (cube_z - 0.01) * scale`
2. **Consider approach angle**: Reward for gripper orientation relative to cube
3. **Increase cube friction**: Make it easier to maintain grip during lift
4. **Try demonstrations**: Use successful IK trajectories as demos for imitation learning

## Files

- `controllers/ik_controller.py` - Damped least-squares IK
- `envs/lift_cube_cartesian.py` - Cartesian action space environment
- `train_lift_cartesian.py` - Training script
- `configs/lift_cartesian_500k.yaml` - Training config
- `eval_cartesian.py` - Evaluation with video recording
