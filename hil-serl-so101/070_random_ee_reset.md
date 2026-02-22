# 070: Random EE Reset for Automated Per-Episode Variation

## Problem

HIL-SERL training needs position variation between episodes for generalization. Previous approach required human to manually reposition the arm each episode, which is tedious and limits training throughput.

## Solution

Add random offset sampling to the existing IK reset mechanism. Each episode, the arm resets to a random position within a configurable box around the base EE target position.

## Implementation

### Config Fields Added (EnvTransformConfig)

```python
random_ee_reset: bool = False  # Enable random offset
random_ee_range_xy: float = 0.03  # ±3cm in x,y
random_ee_range_z: float = 0.02   # ±2cm in z
```

### ResetWrapper Changes

Location: `lerobot/scripts/rl/gym_manipulator.py`

**Before IK Step 3** (after Step 2 wrist positioning):
```python
# Compute reset target (with optional random offset for per-episode variation)
if self.random_ee_reset:
    offset = np.array([
        np.random.uniform(-self.random_ee_range_xy, self.random_ee_range_xy),
        np.random.uniform(-self.random_ee_range_xy, self.random_ee_range_xy),
        np.random.uniform(-self.random_ee_range_z, self.random_ee_range_z)
    ])
    reset_target_ee = self.ik_reset_ee_pos + offset
    logging.info(f"Random EE reset: base={self.ik_reset_ee_pos}, offset={offset}, target={reset_target_ee}")
else:
    reset_target_ee = self.ik_reset_ee_pos
```

**In IK Step 3** - replaced `self.ik_reset_ee_pos` with `reset_target_ee` in:
- Error calculation: `error = np.linalg.norm(reset_target_ee - current_ee)`
- IK computation: `target_joints_rad = self.robot._compute_ik(reset_target_ee, current_joints_rad)`

### Config Example

```json
"wrapper": {
    "use_ik_reset": true,
    "ik_reset_ee_pos": [0.25, 0.0, 0.05],
    "random_ee_reset": true,
    "random_ee_range_xy": 0.03,
    "random_ee_range_z": 0.02,
    "control_time_s": 5.0
}
```

Result: Arm resets to random position in 6cm × 6cm × 4cm box around [0.25, 0.0, 0.05].

## Usage for Grasp-Only PoC

1. Place cube once at start of training session (somewhere near [0.25, 0.0, 0.015])
2. Run training with `configs/grasp_only_hilserl_train_config.json`
3. Each episode:
   - Arm IK resets to random position within box
   - 5 second episode for grasp attempt
   - Reward classifier evaluates success
   - Automatic reset to new random position

No human repositioning needed during training.

## Files Modified

- `lerobot/src/lerobot/envs/configs.py` - Added 3 config fields
- `lerobot/src/lerobot/scripts/rl/gym_manipulator.py` - ResetWrapper random offset logic
- `so101-playground/configs/grasp_only_hilserl_train_config.json` - New training config

## Verification

Check logs show different EE targets each reset:
```
Random EE reset: base=[0.25, 0.0, 0.05], offset=[0.021, -0.015, 0.008], target=[0.271, -0.015, 0.058]
IK reset step 3: Moving to EE target [0.271, -0.015, 0.058]
```
