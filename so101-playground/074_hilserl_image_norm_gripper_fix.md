# HIL-SERL Image Normalization and Gripper Action Fix

## Issue 1: Reward Classifier Not Triggering During Training

**Symptom:** Reward classifier worked in live preview script but never triggered success during HIL-SERL training.

**Root Cause:** Image normalization mismatch between training and reward classifier.

The training config had `normalize_images: false`, meaning images were passed to the reward classifier in [0, 255] range. But the reward classifier expects images in [0, 1] range (normalized).

```json
// configs/grasp_only_hilserl_train_config.json (BEFORE)
"wrapper": {
    "normalize_images": false,  // Images in [0, 255]
    ...
}
```

The reward classifier normalizes inputs using ImageNet stats:
```python
# Classifier.normalize_inputs() expects images in [0, 1]
# Then applies: (img - mean) / std
```

**Fix:** Set `normalize_images: true` in the training config:
```json
// configs/grasp_only_hilserl_train_config.json (AFTER)
"wrapper": {
    "normalize_images": true,  // Images in [0, 1]
    ...
}
```

---

## Issue 2: Gripper Stops Exploring After Episode 1

**Symptom:** Episode 1 showed active gripper open/close. Episode 2+ the gripper closed once and stayed closed, even if it missed the cube.

**Root Cause:** Action space mismatch between SAC policy output and environment expectation.

### The Action Flow

1. **Episode 1 (random sampling):**
   ```python
   # actor.py:573
   action = online_env.action_space.sample()  # Samples from [0, 2] for gripper
   ```

2. **Episode 2+ (policy control):**
   ```python
   # actor.py:554
   action = policy.select_action(batch=obs)  # SAC tanh outputs [-1, 1]
   ```

3. **GripperActionWrapper expected [0, 2]:**
   ```python
   # gym_manipulator.py:1673-1676 (BEFORE)
   gripper_command = action[-1]
   # Gripper actions are between 0, 2
   # we want to quantize them to -1, 0 or 1
   gripper_command = gripper_command - 1.0
   ```

### The Problem

SAC uses tanh squashing which outputs all actions in [-1, 1]. The wrapper always subtracted 1.0, expecting [0, 2] input:

| Policy Output | After -1.0 | Result |
|---------------|------------|--------|
| -1.0 (close)  | -2.0       | Clamped, CLOSE |
| 0.0 (no-op)   | -1.0       | CLOSE (wrong!) |
| 1.0 (open)    | 0.0        | NO ACTION (can't open!) |

The policy could only close the gripper, never open it.

**Fix:** Modified GripperActionWrapper to handle both formats:

```python
# gym_manipulator.py:1673-1678 (AFTER)
gripper_command = action[-1]
# Gripper actions from policy are in [-1, 1] (tanh output)
# -1 = close, 0 = no change, 1 = open
# Handle legacy [0, 2] format from action_space.sample() by converting to [-1, 1]
if gripper_command > 1.0:
    gripper_command = gripper_command - 1.0  # [0, 2] -> [-1, 1]
```

Also updated action stats to match tanh output range:
```json
// configs/grasp_only_hilserl_train_config.json
"action": {
    "min": [-1.0, -1.0, -1.0, -1.0],  // Changed from 0.0
    "max": [1.0, 1.0, 1.0, 1.0]
}
```

---

## Files Changed

### so101-playground
- `configs/grasp_only_hilserl_train_config.json`:
  - `normalize_images: false` -> `true`
  - `action.min[3]: 0.0` -> `-1.0`
  - Updated reward classifier path to v4

### lerobot
- `src/lerobot/scripts/rl/gym_manipulator.py`:
  - GripperActionWrapper now handles [-1, 1] input directly
  - Only converts [0, 2] format when `gripper_command > 1.0`

---

## Verification

After these fixes:
- Reward classifier should trigger when cube is grasped
- Gripper should actively explore open/close in all episodes, not just Episode 1
