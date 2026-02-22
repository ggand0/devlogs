# HIL-SERL Setup for SO-101

## Date: 2026-01-04

## Summary

Set up the prerequisites for Human-in-the-Loop Sample-Efficient RL (HIL-SERL) training on SO-101.

## Key Understanding

**HIL-SERL uses SAC, not DrQ-v2.** This is a fresh policy trained directly on the real robot with:
- Human demonstrations for warm-starting the replay buffer
- Real-time human interventions during training
- Vision-based reward classifier (optional) or manual success annotation

The sim-trained DrQ-v2 policy was useful for validating sim-to-real gap but HIL-SERL takes a different approach.

## What Was Done

### 1. Downloaded SO-101 URDF
```bash
wget -O models/so101_new_calib.urdf \
  https://raw.githubusercontent.com/TheRobotStudio/SO-ARM100/main/Simulation/SO101/so101_new_calib.urdf
```

Key frame name: `gripper_frame_link` (verified from URDF)

### 2. Updated env_config_so101.json

Added to robot config:
- `urdf_path`: Path to URDF for kinematics
- `target_frame_name`: `gripper_frame_link`
- `end_effector_bounds`: Placeholder values (calibrate later)
- `end_effector_step_sizes`: 0.02m per axis

Added to wrapper config:
- `resize_size`: [128, 128]
- `fixed_reset_joint_positions`: null (manual reset or set after calibration)
- Observation flags for velocity/current/EE pose

### 3. Created train_config_so101.json

SAC training config template with:
- Policy: SAC with visual encoder
- Input: gripper_cam (128x128) + state (6 joints)
- Output: 4D action (delta x, y, z + gripper)
- WandB logging enabled

### 4. Installed HIL-SERL Dependencies

```bash
cd /home/gota/ggando/ml/lerobot
uv pip install --system -e ".[hilserl]"
```

Key packages: placo (kinematics), grpcio (actor-learner), gym-hil, wandb

### 5. Implemented SO101FollowerEndEffector in LeRobot

SO-101 was missing end-effector control support (SO-100 had it but SO-101 didn't). Added:

**Files created/modified in lerobot fork:**
- `src/lerobot/robots/so101_follower/config_so101_follower.py`: Added `SO101FollowerEndEffectorConfig` with fields:
  - `urdf_path`: Path to URDF for kinematics
  - `target_frame_name`: End-effector frame (`gripper_frame_link`)
  - `end_effector_bounds`: Workspace limits in meters
  - `end_effector_step_sizes`: Action scaling per axis
  - `max_gripper_pos`: Gripper range limit

- `src/lerobot/robots/so101_follower/so101_follower_end_effector.py`: New class that:
  - Inherits from `SO101Follower`
  - Uses `RobotKinematics` for forward/inverse kinematics via placo
  - Transforms 4D EE-space actions (delta_x, delta_y, delta_z, gripper) to joint-space
  - Clips targets to `end_effector_bounds`

- `src/lerobot/robots/so101_follower/__init__.py`: Export new classes
- `src/lerobot/robots/utils.py`: Register `so101_follower_end_effector` in factory

**Robot type changed from `so101_follower` to `so101_follower_end_effector` in configs.**

## Next Steps (Require Robot)

### Phase 2: Calibrate Workspace
```bash
python find_joint_limits_so101_fixed.py \
  --robot.type=so101_follower_end_effector \
  --robot.port=/dev/ttyACM0 \
  --robot.id=ggando_so101_follower \
  --robot.urdf_path=models/so101_new_calib.urdf \
  --robot.target_frame_name=gripper_frame_link \
  --teleop.type=so101_leader \
  --teleop.port=/dev/ttyACM1 \
  --teleop.id=ggando_so101_leader
```

Move robot through task workspace for 30s, record bounds.

### Phase 3: Record Demonstrations
```bash
cd /home/gota/ggando/ml/lerobot
python -m lerobot.rl.gym_manipulator --config_path /home/gota/ggando/ml/so101-playground/env_config_so101.json
```

Record 10-15 successful pick-and-lift episodes.

### Phase 4: Crop ROI
```bash
python -m lerobot.rl.crop_dataset_roi --repo-id ggando/so101_pick_lift_cube
```

### Phase 5: Train
Terminal 1 (learner):
```bash
python -m lerobot.rl.learner --config_path train_config_so101.json
```

Terminal 2 (actor):
```bash
python -m lerobot.rl.actor --config_path train_config_so101.json
```

### Human Intervention Protocol
- SPACE: Toggle intervention mode
- 's': Mark episode success (reward=1)
- ESC: Mark episode failure (reward=0)
- Leader arm: Guide robot during intervention

## Files Created/Modified

| File | Status |
|------|--------|
| `models/so101_new_calib.urdf` | Created (downloaded) |
| `env_config_so101.json` | Modified |
| `train_config_so101.json` | Created |
| **LeRobot fork changes:** | |
| `lerobot/.../so101_follower/config_so101_follower.py` | Modified (added SO101FollowerEndEffectorConfig) |
| `lerobot/.../so101_follower/so101_follower_end_effector.py` | Created |
| `lerobot/.../so101_follower/__init__.py` | Modified |
| `lerobot/.../robots/utils.py` | Modified (factory) |

## References

- [HIL-SERL Guide](https://huggingface.co/docs/lerobot/hilserl)
- [SO-ARM100 URDF](https://github.com/TheRobotStudio/SO-ARM100/tree/main/Simulation/SO101)
- devlogs/015_hil_serl_handoff.md
