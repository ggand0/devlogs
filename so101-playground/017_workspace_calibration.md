# Workspace Calibration for SO-101

## Date: 2026-01-06

## Summary

Ran workspace calibration using find_joint_limits to get end-effector bounds for HIL-SERL training.

## Prerequisites Fixed

### 1. Missing Dependencies

Installed required packages in lerobot venv:
```bash
cd /home/gota/ggando/ml/lerobot
uv pip install feetech-servo-sdk  # For scservo_sdk motor control
uv pip install placo               # For kinematics (FK/IK)
```

### 2. Uncommitted Changes

Committed gym_manipulator.py fixes to lerobot fork:
- Direct leader-to-follower joint mirroring for robots without URDF
- Added getattr fallbacks for max_gripper_pos config attribute
- Set intervention mode always True for demonstration recording

## Calibration Process

### Script Created

Created `run_find_joint_limits.sh`:
```bash
#!/bin/bash
cd /home/gota/ggando/ml/lerobot

uv run python -m lerobot.scripts.find_joint_limits \
  --robot.type=so101_follower_end_effector \
  --robot.port=/dev/ttyACM0 \
  --robot.id=ggando_so101_follower \
  --robot.urdf_path=/home/gota/ggando/ml/so101-playground/models/so101_new_calib.urdf \
  --robot.target_frame_name=gripper_frame_link \
  --teleop.type=so101_leader \
  --teleop.port=/dev/ttyACM1 \
  --teleop.id=ggando_so101_leader
```

### Results

**End-effector bounds (meters):**
- Max: `[0.185, 0.0051, 0.0096]`
- Min: `[0.1657, -0.0345, -0.0026]`

**Joint position bounds (degrees):**
- Max: `[14.6374, -100.3956, 98.989, 60.6154, 9.5385, 25.5003]`
- Min: `[-2.3297, -103.8242, 97.4066, 52.6154, 6.022, 25.2421]`

### Config Updated

Updated `env_config_so101.json` with calibrated EE bounds:
```json
"end_effector_bounds": {
  "min": [0.1657, -0.0345, -0.0026],
  "max": [0.185, 0.0051, 0.0096]
}
```

## Notes

- URDF self-collision warnings are expected (cosmetic issue in the URDF mesh definitions)
- Current workspace is small (~2cm per axis) - may need recalibration for larger tasks
- The script requires interactive input (calibration prompts) - run in terminal, not from IDE

## Next Steps

1. Record demonstrations using gym_manipulator
2. Crop ROI if needed
3. Start HIL-SERL training (learner + actor)

## Files Modified

| File | Change |
|------|--------|
| `run_find_joint_limits.sh` | Created |
| `env_config_so101.json` | Updated EE bounds |
| lerobot `pyproject.toml` | Added feetech-servo-sdk to deps |
| lerobot `gym_manipulator.py` | Committed SO-101 fixes |
