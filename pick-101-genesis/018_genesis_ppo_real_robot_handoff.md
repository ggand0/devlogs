# Devlog 018: Genesis PPO Agent - Real Robot Deployment Handoff

**Date**: 2025-01-11
**Status**: Ready for deployment testing

## Summary

This document provides all details needed to deploy the Genesis-trained PPO agent on the real SO-101 robot in `so101-playground`. The agent is trained to grasp and lift a cube using wrist camera images + proprioception.

## Training Status

As of 100k steps, the agent successfully grasps and lifts the cube in simulation:
- Episode reward: ~950 (near max)
- Episode length: 300 (full episode, holding cube)
- Success rate: Shows grasping behavior (eval reports 0% due to success definition requiring sustained hold)

**Latest checkpoint**: `runs/ppo_multi_env_wrist/20260111_203814/checkpoints/checkpoint_100352.pt`

---

## Model Architecture

### ActorCritic Network

```
                 +-----------------+
  Image (84x84)  |   CNNEncoder    |  --> (256,) features
       RGB      +-----------------+
                         |
                         v
                    +---------+
  Low-dim (18,) --> | Concat  | --> (320,)
       |            +---------+
       v                  |
  +-----------+           v
  | Low-dim   |    +-------------+
  | Encoder   |    | Actor Mean  | --> (4,) actions
  | (64, 64)  |    +-------------+
  +-----------+    | Actor Std   | --> (4,) std
                   +-------------+
                   |   Critic    | --> (1,) value
                   +-------------+
```

### CNN Encoder (Nature CNN)
```python
Conv2d(3, 32, 8, stride=4)  -> ReLU
Conv2d(32, 64, 4, stride=2) -> ReLU
Conv2d(64, 64, 3, stride=1) -> ReLU
Flatten()                    -> (3136,)
Linear(3136, 256)           -> ReLU -> (256,)
```

### Low-dim Encoder
```python
Linear(18, 64) -> ReLU
Linear(64, 64) -> ReLU -> (64,)
```

---

## Observation Space

### Image Input
- **Shape**: `(3, 84, 84)` - RGB, channels-first
- **Type**: `uint8` (0-255)
- **Normalization**: Done inside CNNEncoder (x / 255.0)
- **Frame stacking**: **NONE** - single frame, unlike DrQ-v2's 3-frame stack

### Low-dim State (18 dims)
| Index | Name | Description |
|-------|------|-------------|
| 0-5 | `joint_pos` | 5 arm joints + 1 gripper (radians) |
| 6-11 | `joint_vel` | 5 arm joints + 1 gripper (rad/s) |
| 12-14 | `gripper_pos` | End-effector XYZ position (meters) |
| 15-17 | `gripper_euler` | End-effector roll, pitch, yaw (radians) |

**Note**: No cube position - the policy is trained to use vision only for object localization.

---

## Action Space

**Shape**: `(4,)` float32 in `[-1, 1]`

| Index | Name | Description |
|-------|------|-------------|
| 0 | `delta_x` | EE X delta (scaled by action_scale) |
| 1 | `delta_y` | EE Y delta (scaled by action_scale) |
| 2 | `delta_z` | EE Z delta (scaled by action_scale) |
| 3 | `gripper` | -1 = closed, +1 = open |

**Action scale**: `0.02` (2cm per unit action per step)

---

## Checkpoint Format

```python
checkpoint = {
    'network_state_dict': OrderedDict,  # ActorCritic weights
    'optimizer_state_dict': OrderedDict,  # Adam optimizer (not needed for inference)
}
```

### Loading Example

```python
import torch
from pathlib import Path

# Minimal ActorCritic for inference (copy from ppo.py or import)
class CNNEncoder(nn.Module):
    def __init__(self, image_channels=3, feature_dim=256):
        super().__init__()
        self.conv = nn.Sequential(
            nn.Conv2d(image_channels, 32, 8, stride=4), nn.ReLU(),
            nn.Conv2d(32, 64, 4, stride=2), nn.ReLU(),
            nn.Conv2d(64, 64, 3, stride=1), nn.ReLU(),
            nn.Flatten(),
        )
        self.fc = nn.Linear(3136, feature_dim)

    def forward(self, x):
        x = x.float() / 255.0  # Normalize
        x = self.conv(x)
        return torch.relu(self.fc(x))

class ActorCritic(nn.Module):
    def __init__(self, image_channels=3, low_dim_size=18, action_dim=4, feature_dim=256):
        super().__init__()
        self.cnn_encoder = CNNEncoder(image_channels, feature_dim)
        self.low_dim_encoder = nn.Sequential(
            nn.Linear(low_dim_size, 64), nn.ReLU(),
            nn.Linear(64, 64), nn.ReLU(),
        )
        combined_dim = feature_dim + 64
        self.actor_mean = nn.Sequential(
            nn.Linear(combined_dim, 256), nn.ReLU(),
            nn.Linear(256, action_dim),
        )
        self.actor_logstd = nn.Parameter(torch.zeros(1, action_dim))
        self.critic = nn.Sequential(
            nn.Linear(combined_dim, 256), nn.ReLU(),
            nn.Linear(256, 1),
        )

# Load checkpoint
checkpoint_path = "runs/ppo_multi_env_wrist/20260111_203814/checkpoints/checkpoint_100352.pt"
checkpoint = torch.load(checkpoint_path, map_location="cuda")

network = ActorCritic().cuda()
network.load_state_dict(checkpoint['network_state_dict'])
network.eval()
```

---

## Inference Code

```python
def get_action(network, rgb: np.ndarray, low_dim: np.ndarray) -> np.ndarray:
    """
    Get deterministic action from policy.

    Args:
        network: Loaded ActorCritic network
        rgb: Image observation (3, 84, 84) uint8
        low_dim: Low-dim state (18,) float32

    Returns:
        action: (4,) float32 in [-1, 1]
    """
    with torch.no_grad():
        # Add batch dim, move to GPU
        image = torch.from_numpy(rgb).unsqueeze(0).cuda()  # (1, 3, 84, 84)
        state = torch.from_numpy(low_dim).unsqueeze(0).float().cuda()  # (1, 18)

        # Forward pass
        img_features = network.cnn_encoder(image)
        low_dim_features = network.low_dim_encoder(state)
        features = torch.cat([img_features, low_dim_features], dim=-1)

        # Get deterministic action (mean, no sampling)
        action = network.actor_mean(features)

        return action.cpu().numpy().squeeze(0)  # (4,)
```

---

## Key Differences from DrQ-v2

| Aspect | DrQ-v2 (pick-101) | PPO (Genesis) |
|--------|-------------------|---------------|
| Frame stacking | 3 frames | 1 frame (no stacking) |
| Image input shape | (3, 3, 84, 84) | (3, 84, 84) |
| Low-dim input shape | (3, 18) | (18,) |
| Architecture | DrQ-v2 encoder | Nature CNN |
| Checkpoint format | RoboBase agent dict | Simple state_dict |
| Loading | Hydra instantiate | Direct load_state_dict |

---

## Wrist Camera Setup

### Simulation Mount
- **Offset from gripper**: `(0.02, -0.08, -0.02)` meters (local frame)
- **Orientation**: Looking down at gripper/object
- **FOV**: 86 degrees

### Real Camera Considerations
- Same camera position/orientation as simulation
- Center crop to square before resize to 84x84
- RGB order (not BGR from OpenCV)

---

## Gripper Action Mapping

In Genesis training, the gripper action is mapped:
```python
GRIPPER_CLOSED = 0.02  # Joint position (rad)
GRIPPER_OPEN = 0.5     # Joint position (rad)

# Action [-1, 1] -> Joint [0.02, 0.5]
gripper_joint = (action + 1) / 2 * (GRIPPER_OPEN - GRIPPER_CLOSED) + GRIPPER_CLOSED
```

For real robot, map the `-1` to `+1` action to your gripper's range.

---

## Initial Pose (Training)

The agent is trained with `curriculum_stage=3`, which starts with:
1. Gripper positioned above cube
2. Wrist joints at pi/2 (top-down orientation)
3. Gripper open

**Initialization sequence**:
```python
# Cube spawn position
CUBE_POS = (0.25, 0.0, 0.015)  # x, y, z

# Pre-grasp offsets
FINGER_WIDTH_OFFSET = -0.015  # Y offset for finger centering
GRASP_Z_OFFSET = 0.005        # Above cube center
HEIGHT_OFFSET = 0.03          # Start 3cm above grasp height

# Initial EE target
initial_target = (
    CUBE_POS[0],
    CUBE_POS[1] + FINGER_WIDTH_OFFSET,
    CUBE_POS[2] + GRASP_Z_OFFSET + HEIGHT_OFFSET
)
# = (0.25, -0.015, 0.05)
```

---

## Control Loop Pseudocode

```python
# Initialize
network = load_checkpoint()
camera = WristCamera()
robot = SO101Robot()
ik = IKController()

# Reset to initial pose (above cube, gripper open, wrist top-down)
robot.move_to_initial_pose()

# Run episode
for step in range(max_steps):
    # Get observation
    rgb = camera.get_frame()  # (H, W, 3) BGR
    rgb = preprocess(rgb)      # crop, resize, BGR->RGB, HWC->CHW -> (3, 84, 84)

    # Get proprioception
    joint_pos = robot.get_joint_positions()  # (6,)
    joint_vel = robot.get_joint_velocities()  # (6,)
    ee_pos = ik.forward_kinematics(joint_pos)[:3]  # (3,)
    ee_euler = ik.forward_kinematics(joint_pos)[3:]  # (3,)

    low_dim = np.concatenate([joint_pos, joint_vel, ee_pos, ee_euler])  # (18,)

    # Get action
    action = get_action(network, rgb, low_dim)  # (4,)

    # Execute
    delta_xyz = action[:3] * action_scale  # Scale delta
    gripper_action = action[3]  # -1 = close, +1 = open

    target_joints = ik.compute_ik(current_pos + delta_xyz, locked_joints=[3, 4])
    robot.send_command(target_joints, gripper_action)

    time.sleep(1.0 / control_hz)
```

---

## Files to Copy/Reference

From `pick-101-genesis`:
1. `src/training/ppo.py` - ActorCritic and CNNEncoder classes
2. `runs/ppo_multi_env_wrist/*/checkpoints/*.pt` - Trained checkpoints

---

## TODO for so101-playground

1. Create `GenesisPPOPolicyRunner` class (or modify `PolicyRunner`)
2. Handle single-frame input (no frame stacking)
3. Test camera preprocessing matches training
4. Verify FK computation matches Genesis (joint order, conventions)
5. Test gripper mapping on real hardware

---

## Quick Test

```bash
# In pick-101-genesis
uv run python src/training/eval_checkpoint.py \
    --checkpoint runs/ppo_multi_env_wrist/20260111_203814/checkpoints/checkpoint_100352.pt
```

---

## Contact

This agent was trained using the Genesis simulator with ray-traced rendering on AMD RX 7900 XTX. The goal is to test if Genesis's more realistic rendering reduces the sim2real gap compared to MuJoCo.
