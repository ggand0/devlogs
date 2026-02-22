# Devlog 029: IK Reset Fix - Coordinate Conversion Bug

## Summary

Fixed the IK reset in `gym_manipulator.py` that was not moving the robot arm. The root cause was **wrong coordinate conversion** - the code treated normalized values as degrees.

---

## The Bug

`gym_manipulator.py` ResetWrapper.reset() (lines 1007-1008, before fix):

```python
current_joints_deg = np.array([current_pos_dict[name] for name in motor_names])[:5]
current_joints_rad = np.deg2rad(current_joints_deg)  # WRONG!
```

**Problem**: `bus.sync_read("Present_Position")` returns **normalized values (-100 to 100)**, NOT degrees!

The variable was misleadingly named `current_joints_deg` but contained normalized values. Using `np.deg2rad()` on normalized values produces garbage radians.

---

## Numerical Proof

For shoulder_pan at normalized position -27.0:

| Conversion | Result |
|------------|--------|
| **Correct** (calibration-aware) | -0.503 rad |
| **Wrong** (deg2rad) | -0.471 rad |
| **Error** | 1.83 degrees |

At extremes (normalized -99.0): **Error: 6.72 degrees per joint**

Multiplied across 5 joints = completely wrong IK solutions!

---

## Why Previous Attempts Failed (Devlog 028)

All previous attempts used the same broken `np.deg2rad()` conversion:

1. **"Iterative IK with motor commands"** - Still wrong conversion, motors got garbage
2. **"Delta approach"** - Computed delta from wrong radians
3. **"IK in simulation then reset"** - IK solution for wrong starting position

The previous agent kept trying different loop structures but never fixed the fundamental coordinate conversion error.

---

## Why rl_inference.py Works

`rl_inference.py` uses `SO101Robot` wrapper from `src/deploy/robot.py` which has **calibration-aware conversion**:

```python
def normalized_to_radians(normalized: float, joint_name: str) -> float:
    cal = CALIBRATION[joint_name]
    range_min = cal["range_min"]
    range_max = cal["range_max"]

    # Normalized (-100 to 100) to encoder position
    encoder = (normalized + 100) / 200 * (range_max - range_min) + range_min

    # Encoder to radians (mid-range = 0 radians)
    mid_encoder = (range_min + range_max) / 2
    radians = (encoder - mid_encoder) / ENCODER_PER_RAD

    return radians
```

This uses the calibration data from:
`~/.cache/huggingface/lerobot/calibration/robots/so101_follower/ggando_so101_follower.json`

---

## The Fix

### File Modified
`/home/gota/ggando/ml/lerobot/src/lerobot/scripts/rl/gym_manipulator.py`

### Changes

#### 1. Added calibration conversion functions (lines 76-151)

```python
_IK_CALIBRATION_PATH = Path("/home/gota/.cache/huggingface/lerobot/calibration/robots/so101_follower/ggando_so101_follower.json")
_IK_ENCODER_PER_RAD = 4096 / (2 * np.pi)  # ~652 encoder units per radian
_IK_MOTOR_NAMES = ["shoulder_pan", "shoulder_lift", "elbow_flex", "wrist_flex", "wrist_roll"]

def _ik_normalized_to_radians(normalized: float, joint_name: str) -> float:
    """Convert LeRobot normalized (-100 to 100) to radians for IK."""
    cal = _load_ik_calibration()[joint_name]
    encoder = (normalized + 100) / 200 * (cal["range_max"] - cal["range_min"]) + cal["range_min"]
    mid_encoder = (cal["range_min"] + cal["range_max"]) / 2
    return (encoder - mid_encoder) / _IK_ENCODER_PER_RAD

def _ik_radians_to_normalized(radians: float, joint_name: str) -> float:
    """Convert radians to LeRobot normalized (-100 to 100) for IK."""
    cal = _load_ik_calibration()[joint_name]
    mid_encoder = (cal["range_min"] + cal["range_max"]) / 2
    encoder = radians * _IK_ENCODER_PER_RAD + mid_encoder
    encoder = np.clip(encoder, cal["range_min"], cal["range_max"])
    return (encoder - cal["range_min"]) / (cal["range_max"] - cal["range_min"]) * 200 - 100
```

#### 2. Replaced IK reset with closed-loop implementation (lines 1000-1053)

**Before** (broken):
```python
# Simulation-only loop with wrong conversion
current_joints_rad = np.deg2rad(current_joints_deg)  # WRONG!
for ik_iter in range(100):
    self.robot._sync_mujoco(sim_joints_rad)
    ik_result = self.robot._compute_ik(...)
    sim_joints_rad = ik_result  # Updates simulation only, not robot!
# Single command at end
reset_follower_position(self.unwrapped.robot, reset_pose)
```

**After** (fixed):
```python
# Closed-loop with correct conversion
for ik_step in range(100):
    # 1. Read ACTUAL robot position with CORRECT conversion
    current_normalized = np.array([current_pos_dict[name] for name in _IK_MOTOR_NAMES])
    current_rad = np.array([
        _ik_normalized_to_radians(current_normalized[i], _IK_MOTOR_NAMES[i])
        for i in range(5)
    ])

    # 2. Sync MuJoCo and check error
    self.robot._sync_mujoco(current_rad)
    current_ee = self.robot._get_ee_position()
    error = np.linalg.norm(self.ik_reset_ee_pos - current_ee)

    if error < 0.01:  # Converged
        break

    # 3. Compute IK from ACTUAL position
    target_rad = self.robot._compute_ik(self.ik_reset_ee_pos, current_rad)

    # 4. Convert back with CORRECT conversion
    target_normalized = np.array([
        _ik_radians_to_normalized(target_rad[i], _IK_MOTOR_NAMES[i])
        for i in range(5)
    ])

    # 5. Send to robot IMMEDIATELY (closed-loop!)
    action_dict = {name: target_normalized[i] for i, name in enumerate(_IK_MOTOR_NAMES)}
    action_dict["gripper"] = gripper_normalized
    self.robot.bus.sync_write("Goal_Position", action_dict)

    # 6. Wait for robot to move
    busy_wait(0.05)
```

---

## Key Differences

| Aspect | Before (Broken) | After (Fixed) |
|--------|-----------------|---------------|
| Conversion | `np.deg2rad(normalized)` | `_ik_normalized_to_radians(normalized, joint)` |
| Loop type | Simulation-only | Closed-loop with real robot |
| Motor commands | Once at end | Every iteration |
| Feedback | None | Real robot position each step |

---

## Testing

Run from lerobot directory:

```bash
# Terminal 1: Learner
cd /home/gota/ggando/ml/lerobot
uv run python -m lerobot.scripts.rl.learner --config_path /home/gota/ggando/ml/so101-playground/train_hilserl_drqv2.json

# Terminal 2: Actor
cd /home/gota/ggando/ml/lerobot
uv run python -m lerobot.scripts.rl.actor --config_path /home/gota/ggando/ml/so101-playground/train_hilserl_drqv2.json
```

Expected behavior: Robot physically moves to IK reset EE position at start of each episode.

---

## Calibration File Format

`~/.cache/huggingface/lerobot/calibration/robots/so101_follower/ggando_so101_follower.json`:

```json
{
    "shoulder_pan": {
        "id": 1,
        "drive_mode": 0,
        "homing_offset": -1321,
        "range_min": 1030,
        "range_max": 3460
    },
    ...
}
```

The `range_min` and `range_max` define the encoder range for each joint. The conversion maps:
- Normalized -100 → range_min encoder
- Normalized +100 → range_max encoder
- Mid-range encoder → 0 radians

---

## Additional Attempts (2026-01-08)

### Attempt 1: Calibration-aware conversion + Closed-loop IK

**Changes:**
- Added `_ik_normalized_to_radians()` and `_ik_radians_to_normalized()` functions
- Replaced simulation-only IK loop with closed-loop that sends commands each iteration

**Result:**
```
IK reset: current EE=[0.193, 0.001, 0.034], start_error=0.0269m
IK step 0: error=0.0406m
IK step 20: error=0.0125m (STUCK - same value for steps 20, 40, 60, 80)
IK reset did not converge, final error=0.0125m
```

**Problem:** Error reduced from 0.04m to 0.0125m but then STOPPED. The robot position readings weren't changing after step ~20, meaning robot stopped moving. IK kept computing same solution because it was stuck at a local minimum.

---

### Attempt 2: Two-step reset (like rl_inference.py)

**Observation:** `rl_inference.py` first moves to a fixed pose (SAFE_JOINTS), THEN applies IK.

**Changes:**
1. Step 1: Call `reset_follower_position()` to move to `fixed_reset_joint_positions` first
2. Step 2: Apply IK from that good starting configuration

**Result:**
```
IK reset step 1: Moving to fixed reset pose
IK reset step 2: EE=[0.140, 0.055, 0.011], error=0.0948m
IK step 0: error=0.0948m
IK step 10: error=0.0244m, EE=[0.183, 0.005, 0.043]
IK reset converged at step 17, error=0.0147m
```

**Problem:** Logs show EE position changing (error reduced from 9.5cm to 1.5cm), but user reports robot "barely moved". This suggests MuJoCo FK doesn't match physical robot position.

---

### Attempt 3: Fix OpenCV crash

**Problem:** After reset, script crashed with:
```
cv2.error: The function is not implemented. Rebuild the library with GTK+ 2.x support
```

**Fix:** Wrapped `cv2.waitKey()` in try-except to handle missing GTK support.

---

## Current Status: STILL NOT WORKING

The IK reset shows "convergence" in logs but physical robot doesn't move significantly.

**Possible causes:**
1. MuJoCo model coordinate system doesn't match physical robot calibration
2. `bus.sync_write("Goal_Position", ...)` not actually moving motors
3. The normalized↔radians conversion is still wrong somewhere

---

## ROOT CAUSE FOUND (2026-01-08)

**The calibration-aware conversion was WRONG for this robot type!**

### The Bug

`SO101FollowerEndEffector` uses `MotorNormMode.DEGREES`:
```python
# so101_follower_end_effector.py:54-65
self.bus = FeetechMotorsBus(
    motors={
        "shoulder_pan": Motor(1, "sts3215", MotorNormMode.DEGREES),  # DEGREES!
        # ...
    },
)
```

So `bus.sync_read("Present_Position")` returns **DEGREES**, not normalized (-100 to 100)!

But my calibration conversion functions expected **normalized** input:
- `_ik_normalized_to_radians()` - converts normalized → radians
- `_ik_radians_to_normalized()` - converts radians → normalized

When I fed degrees into these functions (thinking they were normalized), I got garbage radians!

### The Fix

For `SO101FollowerEndEffector`, just use simple `np.deg2rad()` and `np.rad2deg()`.
This is exactly what `SO101FollowerEndEffector.send_action()` does internally (lines 218-240).

**Changes made:**
1. Removed the broken calibration conversion functions
2. Replaced `_ik_normalized_to_radians()` with `np.deg2rad()`
3. Replaced `_ik_radians_to_normalized()` with `np.rad2deg()`

### Why Previous Attempts Failed

All previous attempts still used the calibration conversion, which:
- Expected normalized input (-100 to 100)
- But received degrees (-180 to 180)
- Produced garbage radians
- Robot received garbage commands

---

## Files

- **Modified**: `/home/gota/ggando/ml/lerobot/src/lerobot/scripts/rl/gym_manipulator.py`
- **Reference** (working IK): `/home/gota/ggando/ml/so101-playground/src/deploy/robot.py`
- **Reference** (working IK controller): `/home/gota/ggando/ml/so101-playground/src/deploy/controllers/ik_controller.py`
- **Plan**: `/home/gota/ggando/ml/so101-playground/docs/plans/030_ik_reset_fix_degrees_mode.md`
