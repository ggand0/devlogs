# SO-101 Image-Based RL Sim2Real Project

## Goal

Train an image-based reinforcement learning policy in simulation (MuJoCo) to grasp and lift a red cube, then deploy it on a real SO-101 robot arm via sim2real transfer.

## Hardware

- **Robot**: SO-101 follower arm (6 DOF + gripper, Feetech STS3215 servos)
- **Camera**: Wrist-mounted USB camera for visual observations
- **Target object**: 3x3x3 cm red cube

## Architecture

### Simulation (pick-101 repo)
- MuJoCo environment with SO-101 URDF/XML model
- DrQ-v2 algorithm (image-based actor-critic with data augmentation)
- Observation: RGB image (84x84) + low-dim state (joint pos/vel, gripper pose)
- Action: Delta XYZ position + gripper command (4-dim)

### Real Robot Deployment (so101-playground repo)
- IK controller converts delta XYZ to joint commands
- Camera preprocessing matches sim (center crop, resize to 84x84)
- State builder provides joint positions and FK-computed gripper pose

## Key Challenges Solved

1. **Kinematic calibration**: Joint offsets between sim and real (~12.5° elbow bias)
2. **Wrist assembly fix**: Physical joint 4-5 connection was 180° flipped
3. **Camera alignment**: Wrist cam pose matching between sim and real
4. **Safe motion**: IK reset sequences to avoid collisions during deployment

## Scripts

| Script | Purpose |
|--------|---------|
| `rl_inference.py` | Deploy DrQ-v2 policy on real robot |
| `ik_grasp_demo.py` | IK-based grasp demo with wrist cam recording |
| `verify_kinematics.py` | Verify FK accuracy between MuJoCo and real robot |

## Status

- Sim training: Working (pick-101 repo)
- Real deployment: In progress (kinematic verification complete, ~1.5cm accuracy)
