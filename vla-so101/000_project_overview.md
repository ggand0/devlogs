# 000 — Project Overview

**vla-so101** — SmolVLA fine-tuning pipeline for SO-101 pick-and-place.

## Goal

Collect teleoperation demos on the SO-101 arm, then fine-tune SmolVLA for autonomous pick-and-place.

## Stack

- **Robot**: SO-101 follower + leader (Feetech STS3215 servos)
- **Camera**: InnoMaker RGB USB (`/dev/video0`)
- **Framework**: [lerobot](https://github.com/huggingface/lerobot) (custom fork, `fix/placo-ee-improvements` branch)
- **IK**: placo (URDF-based FK/IK via `lerobot.model.kinematics.RobotKinematics`)
- **Policy**: SmolVLA (HuggingFace)
- **Package manager**: uv

## Repo structure

```
vla-so101/
├── models/
│   ├── so101_new_calib.urdf    # URDF for placo IK
│   └── assets/                 # STL meshes (gitignored)
├── scripts/
│   ├── record.py               # Data collection with IK reset
│   └── record.sh               # Shell wrapper
├── devlogs/                    # Development logs (gitignored)
├── pyproject.toml
└── uv.lock
```

## Lerobot fork

We use a local editable install of lerobot at `../lerobot` on the `fix/placo-ee-improvements` branch. This branch adds:

- `use_degrees=True` mode for SO101 follower/leader (motor bus reads/writes in degrees)
- `RobotKinematics` class wrapping placo for URDF-based FK/IK (degrees in, degrees out)
- `SO101FollowerEndEffectorConfig` with URDF path, locked joints, EE bounds
- IK helpers: `_clamp_degrees`, `_IK_MOTOR_NAMES`, `_load_ik_calibration`
- `ResetWrapper` in `gym_manipulator.py` with 4-step IK reset

The `pyproject.toml` pulls in lerobot with `[feetech,smolvla,kinematics]` extras.

## Calibration

Calibration files live in `~/.cache/huggingface/lerobot/calibration/`:
- `robots/so101_follower/ggando_so101_follower.json`
- `teleoperators/so101_leader/ggando_so101_leader.json`

These are referenced by `id=` in the robot/teleop configs.
