# Devlog 028: IK Reset - Unsolved Issue

## Overview

IK-based reset for SO-101 robot does NOT work. Despite multiple attempts, the robot does not physically move to IK-computed positions. This devlog documents the problem and all attempted fixes for the next agent.

---

## The Problem

User wants IK reset to move the robot arm to a starting position computed via MuJoCo inverse kinematics. The code computes IK solutions that claim to be correct (error ~0.009m), but the physical robot does NOT move.

---

## Current State of Code

**`lerobot/src/lerobot/scripts/rl/gym_manipulator.py`** - ResetWrapper.reset() (lines 904-992):

```python
if self.use_ik_reset and hasattr(self.robot, '_compute_ik'):
    # Get current joint positions from physical robot
    current_pos_dict = self.robot.bus.sync_read("Present_Position")
    motor_names = list(current_pos_dict.keys())
    current_joints_deg = np.array([current_pos_dict[name] for name in motor_names])[:5]
    current_joints_rad = np.deg2rad(current_joints_deg)
    gripper_deg = current_pos_dict[motor_names[5]] if len(motor_names) > 5 else 50.0

    # Sync MuJoCo and get current EE position
    self.robot._sync_mujoco(current_joints_rad)
    current_ee = self.robot._get_ee_position()

    # Solve IK in simulation (iterate to convergence)
    sim_joints_rad = current_joints_rad.copy()
    for ik_iter in range(100):
        self.robot._sync_mujoco(sim_joints_rad)
        sim_ee = self.robot._get_ee_position()
        error = np.linalg.norm(self.ik_reset_ee_pos - sim_ee)
        if error < 0.01:
            break
        ik_result = self.robot._compute_ik(self.ik_reset_ee_pos, sim_joints_rad)
        sim_joints_rad = ik_result

    # Compute delta from current to IK solution
    total_delta_rad = sim_joints_rad - current_joints_rad
    target_joints_deg = current_joints_deg + np.rad2deg(total_delta_rad)

    reset_pose = np.concatenate([target_joints_deg, [gripper_deg]])

# Then calls:
reset_follower_position(self.unwrapped.robot, reset_pose)
```

**`reset_follower_position`** (lines 76-87):
```python
def reset_follower_position(robot_arm, target_position):
    current_position_dict = robot_arm.bus.sync_read("Present_Position")
    current_position = np.array([current_position_dict[name] for name in current_position_dict])
    trajectory = np.linspace(current_position, target_position, 50)
    for pose in trajectory:
        action_dict = dict(zip(current_position_dict, pose))
        robot_arm.bus.sync_write("Goal_Position", action_dict)
        busy_wait(0.015)
```

---

## Root Cause (Identified but Unsolved)

**MuJoCo model coordinate system does NOT match physical robot calibration.**

The physical robot motors are calibrated with `homing_offset` values stored in:
`~/.cache/huggingface/lerobot/calibration/robots/so101_follower/main_follower/`

Example offsets:
- shoulder_pan: -1321
- shoulder_lift: -1053
- elbow_flex: 899

When IK computes joint positions valid in MuJoCo coordinate space, they produce WRONG physical positions because the MuJoCo model doesn't account for these calibration offsets.

**Evidence:**
- IK logs show error decreasing to ~0.009m (success in simulation)
- `reset_follower_position` is called with computed positions
- Physical robot does NOT move or moves to wrong position
- Dataset first frame positions work: `[-27.0, -99.3, 99.8, 71.1, 1.2, 3.0]`

---

## Attempted Fixes (All Failed)

### 1. Joint Count Mismatch Fix (Committed: cd06aa09)
- **Problem:** `ValueError: shapes (6,) (5,) (5,)` - IK uses 5 arm joints, code was passing 6
- **Fix:** Slice to 5 arm joints, append gripper separately
- **Result:** Fixed the crash, but IK still doesn't move robot

### 2. Tolerance and Target Position Changes
- Changed tolerance from 1cm to 3cm
- Changed target from `[0.25, 0.0, 0.15]` to `[0.20, 0.0, 0.06]`
- **Result:** IK converges faster but robot still doesn't move

### 3. Iterative IK with Motor Commands Each Step
- Tried sending motor commands after each IK iteration
- **Result:** Motors not responding to sync_write during iteration

### 4. IK in Simulation Then reset_follower_position
- Compute full IK solution in simulation first
- Then call reset_follower_position with result
- **Result:** IK claims success, reset_follower_position called, robot doesn't move

### 5. Delta Approach
- Compute delta between current physical position and IK solution
- Apply delta to current position: `target = current + delta`
- **Result:** Still doesn't move robot

### 6. Debug Logging Added
- Added extensive logging showing IK solution, delta, target joints
- Confirmed IK is computing and reset_follower_position is called
- **Result:** Logs look correct but robot doesn't physically move

---

## Sample Log Output (Robot Doesn't Move)

```
INFO IK reset to EE target: [0.2  0.   0.06]
INFO IK reset: current EE=[0.177, 0.001, 0.011], start_error=0.0539m
INFO IK solution: error=0.0091m, delta_deg=[0.189, -3.32, -8.34, -0.04, 0]
INFO IK target joints: [-0.118, -80.55, 79.18, 50.29, -0.044]
INFO Reset the environment.
INFO Reset the environment done.
```

Despite these logs showing success, the physical robot arm does NOT move.

---

## Working Alternative

The `fixed_reset_joint_positions` from config DOES work when IK is disabled:
```json
"fixed_reset_joint_positions": [-27.0, -99.0, 100.0, 71.0, 1.0, 3.0]
```

These values are from the dataset's first frame and produce correct physical positioning.

---

## Config Location

`/home/gota/ggando/ml/so101-playground/train_hilserl_drqv2.json`

Relevant fields:
```json
"wrapper": {
    "fixed_reset_joint_positions": [-27.0, -99.0, 100.0, 71.0, 1.0, 3.0],
    "ik_reset_ee_pos": [0.2, 0.0, 0.06]
}
```

---

## Possible Solutions (Not Yet Attempted)

1. **Calibrate MuJoCo model to match physical robot**
   - Add joint offsets to MuJoCo model that match homing_offset calibration
   - This would make IK output match physical robot coordinates

2. **Apply calibration transform to IK output**
   - After IK computes solution, transform from MuJoCo coords to physical coords
   - Requires understanding the exact relationship between coordinate systems

3. **Use physical robot FK instead of MuJoCo FK**
   - Record multiple physical positions and corresponding EE positions
   - Build a lookup or interpolation for IK

4. **Disable IK reset, use fixed positions**
   - Simply use the working `fixed_reset_joint_positions`
   - User explicitly rejected this option multiple times

---

## Key Files

- `lerobot/src/lerobot/scripts/rl/gym_manipulator.py` - ResetWrapper, reset_follower_position
- `lerobot/src/lerobot/robots/so101_follower/so101_follower_end_effector.py` - _compute_ik, _sync_mujoco
- `train_hilserl_drqv2.json` - Configuration
- Calibration: `~/.cache/huggingface/lerobot/calibration/robots/so101_follower/main_follower/`

---

## User Requirements

User explicitly demanded IK reset to work:
- "DO NOT DISABLE"
- "DO IK RESET"
- "DO NOT EVER DISABLE AT ALL TIME"
- "OBVIOUSLY I WANT FUCKING IK RESET"

---

## Other Learner Error (Separate Issue)

There's also a recurring learner error:
```
AttributeError: 'PolicyFeature' object has no attribute 'get'
```
at `learner.py:1110`:
```python
policy_action_shape = cfg.policy.output_features["action"].get("shape", None)
```

This was supposedly fixed previously but appears to have regressed. The fix should handle both dict and PolicyFeature object.
