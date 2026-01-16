# 067: Cube Detection + State-Based Inference Pipeline

## Date: 2025-01-16

## Goal
Bypass visual sim2real gap by using color-based cube detection to estimate cube position, then feed it to the state-based SAC policy (100% success in sim).

## Architecture
```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│  Wrist Camera   │────▶│  Cube Detector   │────▶│  3D Position    │
│  (640x480 RGB)  │     │  (HSV threshold) │     │  Estimation     │
└─────────────────┘     └──────────────────┘     └────────┬────────┘
                                                          │ cube_xyz
                                                          ▼
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│  Robot Servos   │◀────│  SAC Policy      │◀────│  State Vector   │
│  (6 DOF + grip) │     │  (pretrained)    │     │  (21 dims)      │
└─────────────────┘     └──────────────────┘     └─────────────────┘
                                                   ▲
                                                   │ joint_pos, joint_vel,
                                                   │ gripper_xyz, gripper_euler
                                                ┌──┴──────────────┐
                                                │  Proprioception │
                                                │  (from encoders)│
                                                └─────────────────┘
```

## Implementation

### 1. Cube Detection (`src/perception/cube_detector.py`)
- HSV color thresholding for red cube
- Red wraps around in HSV (0 and 180), so uses two ranges
- Morphological ops (open/close) for noise removal
- Returns center pixel (u,v), bounding box, confidence
- Loads tuned params from `configs/cube_hsv.json`

### 2. Camera Transform (`src/perception/camera_transform.py`)
- Converts 2D pixel coords to 3D world coords
- Uses camera intrinsics from `so101_new_calib.xml`:
  - FOV: 70.5° vertical
  - Position in gripper: [0.008, -0.065, -0.019]
  - Euler in gripper: [0.398, 0, 3.14159]
- Pipeline: pixel → camera ray → gripper frame → world frame
- Intersects ray with table plane (z = cube_half_height)

### 3. HSV Tuning Tool (`scripts/test_perception.py`)
- Live camera feed with detection overlay
- `--tune` mode: OpenCV trackbars for HSV adjustment
- Auto-saves tuned params to `configs/cube_hsv.json` on quit

### 4. Inference Script (`scripts/state_based_inference.py`)
- Imports robot/IK from so101-playground via importlib
- Constructs 21-dim observation:
  - joint_pos [6], joint_vel [6]
  - gripper_xyz [3], gripper_euler [3]
  - cube_xyz [3]
- Loads SAC checkpoint and VecNormalize stats
- Runs at ~20Hz with IK-based Cartesian control

## Tuned HSV Values
From real wrist camera with red cube:
```json
{
  "hsv_lower1": [0, 126, 55],
  "hsv_upper1": [10, 173, 188],
  "hsv_lower2": [160, 126, 55],
  "hsv_upper2": [180, 173, 188]
}
```

## Files Added
- `src/perception/__init__.py`
- `src/perception/cube_detector.py`
- `src/perception/camera_transform.py`
- `scripts/test_perception.py`
- `scripts/state_based_inference.py`
- `configs/cube_hsv.json`

## Dependencies
- Added `opencv-contrib-python>=4.8.0` to pyproject.toml
- Override to exclude `opencv-python-headless` (conflicts with GUI)
- Downgraded numpy<2.0.0 for SAC checkpoint compatibility

## Issues Resolved
- OpenCV GUI not working: pip wheel built without GTK → recreated venv after installing libgtk2.0-dev
- opencv-python-headless conflict: robobase depends on it → added override in pyproject.toml
- Module name conflict: both pick-101 and so101-playground have `src` → use importlib for so101

## Next Steps
1. Test full inference on real robot
2. Verify camera transform accuracy with known cube positions
3. Re-export SAC checkpoint if numpy issues persist
