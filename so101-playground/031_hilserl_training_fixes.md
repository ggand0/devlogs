# Devlog 031: HIL-SERL Training Fixes (2026-01-09)

## Summary

Fixed multiple issues with HIL-SERL training pipeline for SO-101 robot.

---

## Issues Fixed

### 1. Policy Actions Not Being Applied

**Problem**: When not in intervention mode, the robot didn't move - policy actions were ignored.

**Root Cause**: The `RobotEnv.step()` checked for `urdf_path` to determine if EE control was available, but `SO101FollowerEndEffector` uses `mujoco_model_path` instead.

**Fix** (`gym_manipulator.py`):
```python
# BEFORE (line 456):
if not hasattr(self.robot.config, 'urdf_path'):
    # ... fallback to no movement

# AFTER:
has_ee_control = (
    hasattr(self.robot.config, 'mujoco_model_path') and self.robot.config.mujoco_model_path
) or (
    hasattr(self.robot.config, 'urdf_path') and self.robot.config.urdf_path
)

if has_ee_control:
    self.robot.send_action(action_dict)
```

### 2. Intervention Mode Not Moving Follower

**Problem**: In intervention mode ('i' key), follower didn't mirror leader movements.

**Root Cause**: The fix for issue #1 broke intervention - `_leader_positions` was being set but not used because the EE control path was now always taken.

**Fix** (`gym_manipulator.py`):
```python
# Check for leader positions FIRST (joint mirroring takes priority)
leader_positions = getattr(self, '_leader_positions', None)
if leader_positions:
    joint_action = {f"{name}.pos": pos for name, pos in leader_positions.items()}
    self.robot.send_action(joint_action)
    self._leader_positions = None  # Clear after use
else:
    # Normal EE control path
    if has_ee_control:
        self.robot.send_action(action_dict)
```

### 3. Keyboard Input Not Working (Terminal Focus)

**Problem**: Keys ('i', 's', ESC) only worked when OpenCV window was focused, but there's no camera window displayed.

**Root Cause**: `cv2.waitKey()` only captures keys when an OpenCV window is focused.

**Fix**: Enabled pynput for global keyboard capture.

```python
def _init_keyboard_listener(self):
    from pynput import keyboard

    def on_press(key):
        with self.event_lock:
            if key == keyboard.Key.esc:
                self.keyboard_events["episode_end"] = True
            elif hasattr(key, 'char') and key.char == 'k':
                self.keyboard_events["episode_success"] = True
            elif hasattr(key, 'char') and key.char == 'i':
                self._handle_intervention_key()

    self.listener = keyboard.Listener(on_press=on_press)
    self.listener.start()
```

### 4. No Reset Delay for Object Repositioning

**Problem**: No time between episodes to reposition the cube.

**Fix**: Added `reset_delay_s` config option (default 3.0s).

```python
# ResetWrapper.__init__
reset_delay_s: float = 0.0  # Delay after reset for repositioning objects

# After IK reset completes:
if self.reset_delay_s > 0:
    logging.info(f"Waiting {self.reset_delay_s}s for object repositioning...")
    time.sleep(self.reset_delay_s)
```

### 5. IK Reset Target Position Wrong

**Problem**: Initial `ik_reset_ee_pos` was `[0.20, 0.0, 0.12]` which is:
- x=0.20 (cube is at x=0.25)
- y=0.0 (missing finger width offset)
- z=0.12 (12cm - way too high, should be 5cm)

**Root Cause**: Didn't match `rl_inference.py` initial position calculation:
```python
FINGER_WIDTH_OFFSET = -0.015  # Static finger offset from gripper center
GRASP_Z_OFFSET = 0.005
HEIGHT_OFFSET = 0.03  # Start 3cm above grasp height
CUBE_Z = 0.015  # Cube height on table

initial_target = np.array([
    args.cube_x,                              # 0.25
    args.cube_y + FINGER_WIDTH_OFFSET,        # 0.0 - 0.015 = -0.015
    CUBE_Z + GRASP_Z_OFFSET + HEIGHT_OFFSET   # 0.015 + 0.005 + 0.03 = 0.05
])
# Result: [0.25, -0.015, 0.05]
```

**Fix**: Updated default to `[0.25, -0.015, 0.05]` matching rl_inference.py:
- x=0.25 (cube_x)
- y=-0.015 (finger width offset for accurate grip)
- z=0.05 (5cm - ~3cm above cube top, ready to grasp)

---

## Config Changes

Added to `EnvTransformConfig` (`lerobot/src/lerobot/envs/configs.py`):
```python
reset_delay_s: float = 0.0  # Delay after reset for repositioning objects
ik_reset_ee_pos: list[float] | None = None  # Target EE position [x, y, z] for IK reset
```

Updated `train_hilserl_drqv2.json`:
```json
"wrapper": {
    "reset_delay_s": 3.0,
    "ik_reset_ee_pos": [0.25, -0.015, 0.05],
    ...
}
```

---

## Key Bindings (Updated)

- **'i'** - Toggle intervention mode (human teleoperation)
- **'k'** - Mark episode as success (changed from 's')
- **ESC** - End episode
- **Left arrow** - Rerecord episode

---

## Files Modified

- `/home/gota/ggando/ml/lerobot/src/lerobot/scripts/rl/gym_manipulator.py`
  - Fixed EE control detection for MuJoCo-based robots
  - Fixed intervention mode joint mirroring
  - Added pynput global keyboard capture
  - Added reset_delay_s support
  - Changed success key from 's' to 'k'
  - Fixed default ik_reset_ee_pos to [0.25, -0.015, 0.05] (matching rl_inference.py)

- `/home/gota/ggando/ml/lerobot/src/lerobot/envs/configs.py`
  - Added `reset_delay_s` and `ik_reset_ee_pos` fields

- `/home/gota/ggando/ml/so101-playground/train_hilserl_drqv2.json`
  - Added `reset_delay_s: 3.0`
  - Fixed `ik_reset_ee_pos: [0.25, -0.015, 0.05]` (matching rl_inference.py)

- `/home/gota/ggando/ml/so101-playground/scripts/test_ik_reset.py`
  - Updated target to [0.25, -0.015, 0.05] (matching rl_inference.py)

---

## Testing

Confirmed working:
1. Intervention mode ('i') - follower mirrors leader
2. Policy actions - robot moves when not in intervention
3. Success marking ('k') - episode ends with reward 1.0
4. Reset delay - 3 second wait after IK reset
5. Safe return on Ctrl+C - robot lifts and returns to rest
