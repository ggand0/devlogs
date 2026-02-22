# Handoff: HIL-SERL Implementation

## Context

SO-101 robot arm with sim-to-real RL deployment pipeline complete. Pure sim-trained DrQ-v2 policy attempts grasping but fails due to visual/dynamics gap. Next step: Human-in-the-Loop SERL for real-world fine-tuning.

## Current State

### What Works
- **Sim training**: DrQ-v2 in MuJoCo learns lift_cube task (~70% success in sim)
- **Inference pipeline**: `scripts/rl_inference.py` runs policy on real robot
- **IK controller**: MuJoCo-based damped least-squares IK for Cartesian control
- **Camera**: 84x84 RGB, center crop, frame stack matches sim
- **Reset sequence**: Matches training (wrist locked at π/2, -π/2)

### What Doesn't Work
- Policy moves toward cube, attempts grasp, but timing/positioning off
- Visual domain gap: MuJoCo renders ≠ real USB camera
- No domain randomization in current sim training

## Key Files

```
so101-playground/
├── scripts/
│   ├── rl_inference.py      # Main inference loop
│   └── ik_reset_position.py # Calibration tool
├── src/deploy/
│   ├── camera.py            # CameraPreprocessor
│   ├── robot.py             # SO101Robot (LeRobot wrapper)
│   ├── policy.py            # PolicyRunner, LowDimStateBuilder
│   └── controllers/
│       └── ik_controller.py # IKController

pick-101/                     # Sim training repo
├── src/envs/lift_cube.py    # MuJoCo env
├── external/robobase/       # DrQ-v2 implementation
└── runs/image_rl/           # Trained checkpoints
```

## HIL-SERL Overview

Human-in-the-Loop Sample-Efficient RL:
1. Initialize with sim-pretrained policy
2. Collect real robot transitions
3. Human provides sparse reward (success/fail button)
4. Fine-tune with real data + human feedback
5. Iterate until policy works

### Key Components Needed

1. **Replay buffer for real data**
   - Store (obs, action, reward, next_obs, done)
   - Mix with sim data (optional)

2. **Human reward interface**
   - Simple GUI or keyboard input
   - Sparse signal: +1 success, 0 otherwise
   - Optional: -1 for dangerous behavior

3. **Online fine-tuning loop**
   - Collect N transitions
   - Update policy with real data
   - Repeat

4. **Safety constraints**
   - Workspace bounds (already in deploy.yaml)
   - Max joint velocity limits
   - E-stop on Ctrl+C (already implemented)

## Technical Notes

### Robot Interface
```python
from src.deploy.robot import SO101Robot
robot = SO101Robot(port="/dev/ttyACM0")
robot.connect()
joints = robot.get_joint_positions_radians()  # 5 joints
robot.send_action(target_joints, gripper_action)  # gripper: -1=close, 1=open
robot.disconnect()
```

### Policy Interface
```python
from src.deploy.policy import PolicyRunner
policy = PolicyRunner(checkpoint_path, device="cuda")
policy.load()
action = policy.get_action(rgb_obs, low_dim_obs)  # Returns 4D: [dx, dy, dz, gripper]
```

### IK Interface
```python
from src.deploy.controllers import IKController
ik = IKController()
ik.sync_joint_positions(current_joints)
target_joints = ik.cartesian_to_joints(delta_xyz, current_joints, action_scale=0.02)
```

### Observation Shapes
- `rgb_obs`: (3, 3, 84, 84) - frame_stack=3, CHW format, uint8
- `low_dim_obs`: (3, 17) - frame_stack=3, proprioception
- `action`: (4,) - [dx, dy, dz, gripper] in [-1, 1]

## References

- [SERL Paper](https://arxiv.org/abs/2401.16013) - Sample-Efficient RL
- [HIL-SERL](https://hil-serl.github.io/) - Human-in-the-Loop variant
- [RoboBase](https://github.com/SamsungLabs/robobase) - DrQ-v2 implementation used

## Next Steps

1. Design human reward interface (keyboard? simple GUI?)
2. Implement real replay buffer
3. Add online update loop to inference script
4. Test with 10-20 real episodes
5. Iterate on reward signal design
