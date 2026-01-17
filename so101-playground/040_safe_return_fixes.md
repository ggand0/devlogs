# Devlog 040: Safe Return Sequence Fixes

## Problem

The `safe_return()` function in `rl_inference.py` was causing motor overload errors when interrupted with Ctrl+C. The robot would fail to return to rest position and throw errors like:

```
RuntimeError: Failed to write 'Torque_Enable' on id_=4 with '0' after 6 tries. [RxPacketError] Overload error!
```

### Root Causes

1. **Sudden wrist movement**: When wrists are at π/2 (top-down orientation) and the code tries to send `SAFE_JOINTS` (all zeros), the wrists attempt a sudden 90-degree movement which overloads the motors.

2. **Port busy after Ctrl+C**: Interrupting during serial communication leaves the port in a busy state, causing subsequent commands to fail.

3. **Direct jump to rest**: Going directly from any position to `REST_JOINTS` without intermediate steps causes sudden movements.

## Solution

Updated `safe_return()` to use a 2-step approach:

### Step 1: Lift with IK
Use IK to lift the end-effector to a safe height (15cm) while keeping wrists locked at π/2. This maintains the current wrist orientation during the lift.

### Step 2: Interpolate to REST_JOINTS
Instead of jumping directly to `REST_JOINTS`, interpolate over 20 steps (2 seconds total). This allows the wrists to gradually move from π/2 to their rest position without overloading.

### Additional Fixes

1. **Serial port recovery delay**: Added 0.5s delay before `safe_return()` to let the serial port recover from interrupted communication.

2. **Exception handling**: Wrapped `safe_return()` in try/except to handle any failures gracefully.

3. **Updated REST_JOINTS**: Updated rest position to current calibrated values: `[-0.0591, -1.8415, 1.7135, 0.7210, -0.1097]`

## Code Changes

### rl_inference.py

```python
def safe_return():
    """Safe return sequence: lift up first, then go to rest position."""
    print("\nSafe return sequence...")

    # Step 1: Lift up to safe height (keep wrist orientation)
    print("  Lifting to safe height...")
    try:
        current_joints_raw = robot.get_joint_positions_radians()
        if use_genesis:
            ik.sync_joint_positions(apply_joint_offset(current_joints_raw))
        else:
            ik.sync_joint_positions(current_joints_raw)
        current_ee = ik.get_ee_position()
        safe_height_target = current_ee.copy()
        safe_height_target[2] = 0.15  # Lift to 15cm

        for step in range(40):
            # IK loop with wrist locked at π/2
            ...

    except Exception as e:
        print(f"  Warning: Failed to lift ({e}), going directly to rest...")

    # Step 2: Interpolate to rest position (gradual wrist movement)
    print("  Returning to rest position...")
    current_joints = robot.get_joint_positions_radians()
    for i in range(20):
        alpha = (i + 1) / 20
        interp_joints = (1 - alpha) * current_joints + alpha * REST_JOINTS
        robot.send_action(interp_joints, -1.0)
        time.sleep(0.1)
    robot.send_action(REST_JOINTS, -1.0)
    time.sleep(1.0)
```

### Cleanup block

```python
finally:
    print("\nCleaning up...")
    # ... camera cleanup ...

    # Wait for serial port to recover from interrupted communication
    time.sleep(0.5)

    try:
        safe_return()
    except Exception as e:
        print(f"  Warning: safe_return failed ({e})")

    robot.disconnect()
    print("Done.")
```

## Key Insights

1. **Wrist joint limits**: The wrist motors can't handle sudden large movements. Always interpolate when moving more than ~30 degrees.

2. **Serial port contention**: After Ctrl+C interrupts a serial communication, the port needs time to recover. A 0.5s delay is usually sufficient.

3. **IK for safe movements**: Using IK to lift maintains the current wrist orientation, avoiding the need to change wrist joints during the lift phase.

4. **Interpolation timing**: 20 steps at 0.1s each (2 seconds total) provides smooth movement without being too slow.
