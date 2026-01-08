# Devlog 030: DEGREES Mode Negative Encoder Bug

## Summary

Extreme degree values in DEGREES mode produce negative encoder values, causing `ValueError: Negative values are not allowed` in the motor bus.

---

## The Bug

`motors_bus.py` DEGREES mode conversion (line 826-829):

```python
mid = (min_ + max_) / 2
max_res = 4095
unnormalized_values[id_] = int((val * max_res / 360) + mid)
```

When `val` (degrees) is extremely negative, the encoder value goes negative:

```
encoder = (degrees * 4095 / 360) + mid
```

For `encoder < 0`:
```
degrees < -mid * 360 / 4095
```

---

## Numerical Proof (shoulder_lift)

Calibration: `range_min=827, range_max=3225, mid=2026`

| Degrees | Encoder | Result |
|---------|---------|--------|
| -99° | 901 | OK |
| -150° | 320 | OK (but beyond physical limit) |
| -178° | ~0 | Threshold |
| -200° | -249 | **ERROR** |
| -500° | -3560 | **ERROR** |

Threshold for negative encoder: **~-178°** for shoulder_lift.

---

## Error Message

```
ValueError: Negative values are not allowed: -148
```

At `motors_bus.py:854`:
```python
def _serialize_data(self, value: int, length: int) -> list[int]:
    ...
    raise ValueError(f"Negative values are not allowed: {value}")
```

---

## When This Happens

1. MuJoCo IK computes joint angle beyond calibrated range
2. DEGREES mode converts to negative encoder
3. Bus rejects with ValueError
4. Robot doesn't move

---

## The Fix

Clamp degree values to valid encoder range (0 to 4095) before sending to bus.

For each joint, compute degree limits:
- `min_deg = -mid * 360 / 4095` (encoder = 0)
- `max_deg = (4095 - mid) * 360 / 4095` (encoder = 4095)

Then clamp within calibration range:
- `min_deg = max(min_deg, (range_min - mid) * 360 / 4095)`
- `max_deg = min(max_deg, (range_max - mid) * 360 / 4095)`

---

## Safe Degree Limits (preventing negative encoder)

| Joint | mid | min_deg (enc=0) | max_deg (enc=4095) |
|-------|-----|-----------------|-------------------|
| shoulder_pan | 2245 | -197.4° | +162.6° |
| shoulder_lift | 2026 | -178.1° | +181.9° |
| elbow_flex | 2043 | -179.6° | +180.4° |
| wrist_flex | 2129.5 | -187.2° | +172.8° |
| wrist_roll | 2021 | -177.7° | +182.3° |

---

## Files Modified

- `/home/gota/ggando/ml/lerobot/src/lerobot/scripts/rl/gym_manipulator.py`
  - Updated `_get_degree_limits()` to also prevent negative encoder values

---

## Three-Step IK Reset (2026-01-08)

### Problem

The IK reset in `gym_manipulator.py` didn't match `rl_inference.py`. It used `fixed_reset_joint_positions = [-27, -99, 100, 71, 1]` degrees as the starting pose, but `rl_inference.py` uses `SAFE_JOINTS = np.zeros(5)` (all zeros radians).

### rl_inference.py Pattern

```python
# Step 1: Reset to safe extended position
SAFE_JOINTS = np.zeros(5)  # All zeros
robot.send_action(SAFE_JOINTS, 1.0)
time.sleep(1.5)

# Step 2: Set wrist joints to π/2 for top-down orientation
topdown_joints = robot.get_joint_positions_radians().copy()
topdown_joints[3] = np.pi / 2   # wrist_flex = 90°
topdown_joints[4] = -np.pi / 2  # wrist_roll = -90°
robot.send_action(topdown_joints, 1.0)
time.sleep(1.0)

# Step 3: Move to training initial position with IK
move_to_initial_pose_with_wrist_lock(robot, ik, initial_target)
```

### The Fix

Updated `gym_manipulator.py` ResetWrapper.reset() to use the same three-step sequence:

1. **Step 1**: Move to SAFE_JOINTS (all zeros degrees)
2. **Step 2**: Set wrist_flex=90°, wrist_roll=-90° for top-down orientation
3. **Step 3**: Closed-loop IK to reach target EE position

### Test Results

```
=== THREE-STEP IK RESET TEST ===

Starting position: ['-3.5', '-85.8', '89.8', '69.6', '0.5']

STEP 1: Moving to SAFE_JOINTS (all zeros)...
  Reached: ['-0.8', '0.8', '5.5', '1.7', '0.5']

STEP 2: Setting wrist orientation (90, -90)...
  Reached: ['-0.8', '2.3', '7.0', '89.8', '-89.5']

STEP 3: Moving to target EE position [0.2, 0.0, 0.06]...
  Start: EE=[0.21049294 0.0023245  0.05401225], error=0.0123m
  Converged at step 0, error=0.0123m
  Final: ['-0.8', '2.3', '7.0', '89.8', '-89.5']

=== TEST COMPLETE ===
```

The three-step reset now matches `rl_inference.py` and converges within 1.23cm error immediately after step 2.

---

## Test Script

Added `./scripts/test_ik_reset.py` to verify the three-step IK reset works correctly.

Usage:
```bash
cd /home/gota/ggando/ml/lerobot
uv run python /home/gota/ggando/ml/so101-playground/scripts/test_ik_reset.py
```

---

## Safe Return on Ctrl+C (2026-01-08)

### Problem

When the actor script is interrupted (ctrl+c), the robot stays in its current position which may be unsafe. The `rl_inference.py` script has a `safe_return()` function that lifts the arm and returns to a rest position.

### The Fix

Added `safe_return_to_home()` function to `actor.py` matching `rl_inference.py` behavior:

1. **Step 1**: Lift end-effector to safe height (15cm) using IK with locked wrist orientation
2. **Step 2**: Return to REST_JOINTS position

The main loop in `act_with_policy()` is wrapped in try/finally to ensure safe_return is called on any exit (ctrl+c, shutdown_event, normal completion).

### Code Changes

**File:** `/home/gota/ggando/ml/lerobot/src/lerobot/scripts/rl/actor.py`

```python
# Added constants
SAFE_JOINTS_RAD = np.zeros(5)
REST_JOINTS_RAD = np.array([-0.2424, -1.8040, 1.6582, 0.7309, -0.0629])

# Added function
def safe_return_to_home(online_env):
    """Safe return sequence: lift up first, then go to rest position."""
    # Step 1: Lift to 15cm height with locked wrist
    # Step 2: Return to REST_JOINTS position

# Modified act_with_policy()
try:
    for interaction_step in range(cfg.policy.online_steps):
        # ... main loop ...
finally:
    safe_return_to_home(online_env)
```
