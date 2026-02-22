# Devlog 027: HIL-SERL Actor Fixes

## Overview

Fixed remaining issues preventing HIL-SERL actor from running correctly with the SO-101 robot, including action conversion for the last frame, keyboard input capture issues, and IK-based reset.

---

## Fix 1: Last Frame Action Conversion

### Problem

Buffer conversion crashed on the last frame when converting joint actions (6-dim) to EE actions (4-dim):
```
Action conversion failed on the last frame because it used raw joint actions without conversion
```

### Root Cause

The last frame handling in `buffer.py` used `prev_sample["action"]` directly without applying the action conversion logic that was applied to all other frames.

### Solution

Added action conversion for the last frame with zero EE delta (since there's no next sample to compute delta from):

**`lerobot/src/lerobot/utils/buffer.py`:**
```python
action = prev_sample["action"]

# Convert joint actions to EE delta actions for the last frame
# Since there's no next sample, use zero EE delta
if convert_actions_to_ee and mj_model is not None:
    # Gripper action: use gripper value from action
    gripper_val = action[-1].item()
    gripper_normalized = (gripper_val / 50.0) - 1.0  # 0->-1, 100->1
    gripper_normalized = np.clip(gripper_normalized, -1.0, 1.0)

    # Zero EE delta for last frame (no movement)
    ee_action = np.array([0.0, 0.0, 0.0, gripper_normalized], dtype=np.float32)
    action = torch.from_numpy(ee_action)

action = action.unsqueeze(0).to(storage_device)
```

---

## Fix 2: Keyboard Capturing Terminal Input

### Problem

User typing in terminal was triggering keyboard events in the actor:
```
Space key pressed
Human intervention step triggered
```

The user clarified: "ITS NOT A Phantom key presses BITCH" - they were typing in the terminal and pynput was capturing it globally.

### Root Cause

`pynput` captures keyboard input globally (system-wide), not just from the application window.

### Solution

Changed from pynput to cv2.waitKey which only captures input when the OpenCV window is focused:

**`lerobot/src/lerobot/scripts/rl/gym_manipulator.py`:**
```python
def _init_keyboard_listener(self):
    """Initialize keyboard handling via cv2.waitKey (window-focused only)."""
    self.listener = None  # No pynput listener - use cv2 instead

def _poll_cv2_keys(self):
    """Poll for keyboard input via cv2.waitKey."""
    import cv2
    key = cv2.waitKey(1) & 0xFF
    if key == 255:  # No key pressed
        return
    with self.event_lock:
        if key == 27:  # ESC
            self.keyboard_events["episode_end"] = True
        elif key == ord('s'):
            logging.info("Key 's' pressed. Episode success triggered.")
            self.keyboard_events["episode_success"] = True
        elif key == 81:  # Left arrow
            self.keyboard_events["rerecord_episode"] = True
        elif key == ord(' '):  # Space
            self._handle_space_key()
```

Also added `_handle_space_key()` method for subclass override in `GearedLeaderControlWrapper`.

---

## Fix 3: Intervention Mode Always True

### Problem

Robot was always in human intervention mode regardless of keyboard state.

### Root Cause

`_check_intervention()` in `GearedLeaderControlWrapper` was returning `True` unconditionally.

### Solution

Fixed to return the actual keyboard toggle state:

```python
def _check_intervention(self):
    """Check if human intervention is active based on keyboard toggle."""
    return self.keyboard_events["human_intervention_step"]
```

---

## Fix 4: IK-Based Reset Position

### Problem

User complained: "use the IK reset pose, it's resetting to shitty position" and "why do I have to move the arm manually to reset even?"

The fixed joint position reset wasn't putting the arm in a good starting position.

### Solution

Added IK-based reset that computes joint positions from target EE position:

**`lerobot/src/lerobot/scripts/rl/gym_manipulator.py`:**
```python
class ResetWrapper(gym.Wrapper):
    def __init__(
        self,
        env: RobotEnv,
        reset_pose: np.ndarray | None = None,
        reset_time_s: float = 5,
        use_ik_reset: bool = False,
        ik_reset_ee_pos: list | None = None,
    ):
        # ...
        self.use_ik_reset = use_ik_reset
        self.ik_reset_ee_pos = np.array(ik_reset_ee_pos) if ik_reset_ee_pos else np.array([0.25, 0.0, 0.15])
        self._ik_reset_pose = None  # Cached IK-computed reset pose

    def reset(self, ...):
        if self.use_ik_reset and hasattr(self.robot, '_compute_ik'):
            if self._ik_reset_pose is None:
                # Get current joint positions
                current_pos_dict = self.robot.bus.sync_read("Present_Position")
                current_joints_deg = np.array([current_pos_dict[name] for name in current_pos_dict])[:6]
                current_joints_rad = np.deg2rad(current_joints_deg)

                # Compute IK for target EE position
                self.robot._sync_mujoco(current_joints_rad)
                target_joints_rad = self.robot._compute_ik(self.ik_reset_ee_pos, current_joints_rad)
                self._ik_reset_pose = np.rad2deg(target_joints_rad)
                logging.info(f"Computed IK reset pose for EE target {self.ik_reset_ee_pos}: {self._ik_reset_pose}")
            reset_pose = self._ik_reset_pose
```

Auto-enabled IK reset when robot has `mujoco_model_path`:

```python
use_ik_reset = hasattr(cfg.robot, 'mujoco_model_path') and cfg.robot.mujoco_model_path is not None
```

---

## Commits

| Hash | Description |
|------|-------------|
| `4765b00d` | fix(rl): convert joint actions to EE actions including last frame |
| `9c0e3863` | fix(rl): use cv2.waitKey instead of pynput for window-focused keyboard input |
| `6336d682` | feat(rl): add IK-based reset for SO101FollowerEndEffector |

---

## Files Changed

- `lerobot/src/lerobot/utils/buffer.py` - Last frame action conversion
- `lerobot/src/lerobot/scripts/rl/gym_manipulator.py` - cv2.waitKey keyboard, intervention mode fix, IK reset

---

## Usage

```bash
# Run learner (first terminal)
cd /home/gota/ggando/ml/so101-playground
uv run python -m lerobot.scripts.rl.learner --config_path train_hilserl_drqv2.json

# Run actor (second terminal)
cd /home/gota/ggando/ml/so101-playground
uv run python -m lerobot.scripts.rl.actor --config_path train_hilserl_drqv2.json
```

Keyboard controls (only work when camera window is focused):
- **Space**: Toggle human intervention mode
- **S**: Mark episode as success
- **Left Arrow**: Rerecord episode
- **ESC**: End episode

---

## Related

- Devlog 026: HIL-SERL Actor-Learner Integration Fixes
- Devlog 025: HIL-SERL Learner Fixes
