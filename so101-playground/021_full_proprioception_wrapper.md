# Devlog 021: Full Proprioception Wrapper for DrQ-v2 Sim-to-Real

## Overview

Implemented the `FullProprioceptionWrapper` to provide 54-dim proprioceptive state on the real robot, matching the corrected sim DrQ-v2 training checkpoint.

## Background

The sim DrQ-v2 training was initially producing checkpoints with 63-dim low_dim_state that included privileged `cube_pos` information (not available on real robot). After fixing the sim side to remove cube_pos, the new checkpoint expects 54-dim state:

```
actor_model.input_preprocess_modules.low_dim_obs.0.weight: torch.Size([50, 54])
```

This is 18 dims per frame × 3 frame stack = 54 dims.

## State Composition

Each frame contains 18 dimensions:

| Component | Dims | Source |
|-----------|------|--------|
| joint_pos | 6 | Direct from motor encoders |
| joint_vel | 6 | Computed from position delta / dt |
| gripper_xyz | 3 | Forward kinematics (FK) |
| gripper_euler | 3 | FK rotation matrix → euler angles |

## Implementation

### 1. FullProprioceptionWrapper (`gym_manipulator.py:1183-1297`)

```python
class FullProprioceptionWrapper(gym.ObservationWrapper):
    """
    Wrapper that provides the full 18-dim proprioceptive state matching
    the sim DrQ-v2 training.
    """

    def __init__(self, env, fps=30, num_dof=6):
        # Initialize kinematics for FK computation
        self.kinematics = RobotKinematics(
            urdf_path=env.unwrapped.robot.config.urdf_path,
            target_frame_name=env.unwrapped.robot.config.target_frame_name,
        )

    def observation(self, observation):
        # Get current joint positions (6 dims)
        joint_pos = observation["agent_pos"][:self.num_dof]

        # Compute joint velocities (6 dims)
        joint_vel = (joint_pos - self.last_joint_positions) / self.dt

        # Get end-effector pose via FK
        T = self.kinematics.forward_kinematics(current_joint_pos)
        ee_xyz = T[:3, 3]  # Position (3 dims)
        ee_euler = rotation_matrix_to_euler(T[:3, :3])  # Orientation (3 dims)

        # Concatenate into full 18-dim state
        full_state = np.concatenate([joint_pos, joint_vel, ee_xyz, ee_euler])
        observation["agent_pos"] = full_state
        return observation
```

### 2. Euler Angle Extraction (`gym_manipulator.py:1156-1180`)

Uses ZYX convention (yaw-pitch-roll) common in robotics:

```python
def rotation_matrix_to_euler(R):
    sy = np.sqrt(R[0, 0] ** 2 + R[1, 0] ** 2)
    singular = sy < 1e-6

    if not singular:
        roll = np.arctan2(R[2, 1], R[2, 2])
        pitch = np.arctan2(-R[2, 0], sy)
        yaw = np.arctan2(R[1, 0], R[0, 0])
    else:
        # Gimbal lock case
        roll = np.arctan2(-R[1, 2], R[1, 1])
        pitch = np.arctan2(-R[2, 0], sy)
        yaw = 0

    return np.array([roll, pitch, yaw])
```

### 3. Config Option (`configs.py:182`)

Added to `EnvTransformConfig`:

```python
add_full_proprioception: bool = False  # 18-dim state: joint_pos + joint_vel + ee_xyz + ee_euler
```

### 4. Factory Integration (`gym_manipulator.py:1947-1960`)

```python
if cfg.wrapper:
    if getattr(cfg.wrapper, 'add_full_proprioception', False):
        env = FullProprioceptionWrapper(env=env, fps=cfg.fps)
    else:
        # Legacy mode: individual wrappers
        if cfg.wrapper.add_joint_velocity_to_observation:
            env = AddJointVelocityToObservation(env=env, fps=cfg.fps)
        # ...
```

## Configuration

Updated `train_hilserl_drqv2.json`:

```json
{
  "policy": {
    "input_features": {
      "observation.images.gripper_cam": {
        "type": "VISUAL",
        "shape": [9, 84, 84]
      },
      "observation.state": {
        "type": "STATE",
        "shape": [54]  // Changed from [6]
      }
    }
  },
  "env": {
    "robot": {
      "urdf_path": "/home/gota/ggando/ml/so101-playground/models/so101_new_calib.urdf",
      "target_frame_name": "gripper_frame_link"
    },
    "wrapper": {
      "add_full_proprioception": true  // Enable 18-dim state
    }
  }
}
```

## Dependencies

- **placo**: Used for forward kinematics computation via `RobotKinematics`
- **URDF**: Required for FK - must provide `urdf_path` and `target_frame_name` in robot config

## Frame Stack

DrQ-v2 uses frame stacking for temporal information:
- `frame_stack: 3` in policy config
- Real robot wrapper provides 18-dim per frame
- Policy receives 54-dim stacked state (handled by frame stack wrapper in training pipeline)

## Sim-Real State Matching

| Dimension | Sim (pick-101) | Real (so101-playground) |
|-----------|----------------|-------------------------|
| 0-5 | joint_pos | joint_pos (from encoders) |
| 6-11 | joint_vel | joint_vel (position delta) |
| 12-14 | gripper_xyz | gripper_xyz (FK) |
| 15-17 | gripper_euler | gripper_euler (FK) |

The sim uses MuJoCo's joint sensors and FK, while real uses motor encoders and placo FK.

## Files Changed

- `lerobot/src/lerobot/scripts/rl/gym_manipulator.py`
  - Added `rotation_matrix_to_euler()` function
  - Added `FullProprioceptionWrapper` class
  - Updated `make_robot_env()` factory
- `lerobot/src/lerobot/envs/configs.py`
  - Added `add_full_proprioception` option to `EnvTransformConfig`
- `so101-playground/train_hilserl_drqv2.json`
  - Updated state shape to [54]
  - Added URDF path and target frame
  - Enabled `add_full_proprioception`

## Next Steps

1. Wait for sim training with 54-dim state to complete at `pick-101/runs/image_rl/20260106_221053`
2. Convert the checkpoint to LeRobot format
3. Test DrQ-v2 policy on real SO-101 robot with HIL-SERL fine-tuning
