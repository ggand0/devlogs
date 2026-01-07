# DrQ-v2 Action Saturation Debug

## Problem
DrQ-v2 policy outputs saturated actions (±1) on real robot, causing erratic movement away from cube and hitting the ground.

## Root Cause Analysis
The sim-trained policy expects specific state formats that differ from what the LeRobot gym wrapper provides.

## Fixes Applied

### 1. Wrist Roll Sign Flip
**File:** `scripts/eval_drqv2.py` - `normalize_state()`

Real robot and MuJoCo sim have inverted wrist_roll direction. Found in `rl_inference.py`:
```python
topdown_joints[4] = -np.pi / 2  # wrist_roll (flipped for real robot)
```

Fix: Negate wrist_roll position (index 4) and velocity (index 10).

### 2. Gripper Conversion to Radians
**File:** `scripts/eval_drqv2.py` - `normalize_state()`

Sim uses raw MuJoCo gripper joint angle in radians `[-0.17, 1.75]`, not normalized `[-1, 1]`.

Old (wrong):
```python
state[..., 5] = (state[..., 5] / 50.0) - 1.0  # [-1, 1]
```

New (correct):
```python
GRIPPER_RAD_MIN = -0.17453
GRIPPER_RAD_MAX = 1.74533
GRIPPER_RAD_RANGE = GRIPPER_RAD_MAX - GRIPPER_RAD_MIN
state[..., 5] = (state[..., 5] / 100.0) * GRIPPER_RAD_RANGE + GRIPPER_RAD_MIN
```

### 3. MuJoCo FK for ee_pose
**File:** `scripts/eval_drqv2.py` - `build_state_with_mujoco_fk()`

LeRobot uses Pinocchio/URDF kinematics which produces different ee_euler values than MuJoCo:
- LeRobot ee_euler: `[-2.94, 0.008, 1.47]`
- MuJoCo ee_euler: `[-1.53, 1.37, 3.09]`

Fix: Replace ee_xyz and ee_euler with MuJoCo FK computed values.

## Current State After Fixes
Debug output shows normalized state values are within expected ranges:
```
shoulder_pan: [-1.92, 1.92] rad, got 0.056
shoulder_lift: [-1.75, 1.75] rad, got 0.330
elbow_flex: [-1.69, 1.69] rad, got -0.126
wrist_flex: [-1.66, 1.66] rad, got 1.569
wrist_roll: [-2.74, 2.84] rad, got 1.563
gripper: [-0.17, 1.75] rad, got 1.70
```

**But actions are still saturated:**
```
action_logits (pre-tanh): [-18.81, 8.60, 17.36, -1.20]
action (post-tanh): [-1.0, 1.0, 1.0, -0.83]
```

## Remaining Issues to Investigate

1. **ee_euler mismatch**: MuJoCo ee_euler `[-1.53, 1.37, 3.09]` may not match what sim training produced for the same physical pose. Need to verify euler angle computation matches sim exactly.

2. **Observation format**: Sim `LiftCubeCartesianEnv` has 21-dim obs (includes cube_pos), but policy expects 18-dim. Need to verify how cube_pos was removed during training.

3. **Action coordinate frame**: Actions [dx, dy, dz] might be in different coordinate frames between sim and real.

4. **Other joint sign conventions**: Only wrist_roll was identified as flipped. Other joints might also have sign issues.

## Key Files
- `scripts/eval_drqv2.py` - Main evaluation script with state normalization
- `src/deploy/controllers/ik_controller.py` - MuJoCo FK computation
- `scripts/rl_inference.py` - Reference for real robot state building
- `/home/gota/ggando/ml/pick-101/src/envs/lift_cube_cartesian.py` - Sim environment

## Debug Command
```bash
python scripts/eval_drqv2.py --config train_hilserl_drqv2.json --use_ik_reset --action_scale 0.05 --debug
```
