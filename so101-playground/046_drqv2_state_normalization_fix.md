# Devlog 046: DrQ-v2 State Unit Mismatch Fix

**Date**: 2025-01-19
**Status**: Complete

## Problem

Robot arm exhibited constant shaking during HIL-SERL training with DrQ-v2. Both DrQ-v2 with pretrained weights AND SAC from scratch had the exact same shaking behavior, indicating the issue was in the shared infrastructure, not policy-specific.

## Root Cause Analysis

### Initial Hypothesis (WRONG)
Initially thought DrQV2Policy was missing input normalization. Added `normalize_inputs` calls similar to other policies (ACT, TDMPC, Diffusion, SAC).

### Actual Root Cause: Unit Mismatch

**RoboBase/Genesis uses IDENTITY normalization (NO normalization) and expects RADIANS.**

From sim training team's analysis:
```python
# RoboBase's _act_extract_low_dim_state() - NO normalization!
def _act_extract_low_dim_state(self, observations):
    low_dim_obs = extract_from_spec(observations, "low_dim_state")
    if self.frame_stack_on_channel:
        low_dim_obs = flatten_time_dim_into_channel_dim(low_dim_obs)
    return low_dim_obs  # Raw values, no normalization
```

### Expected vs Actual State Format

| Component | Genesis (Expected) | Real Robot (Actual) | Issue |
|-----------|-------------------|---------------------|-------|
| Joint positions | **RADIANS** (-π to π) | **DEGREES** (-180 to 180) | 57x scale mismatch |
| Joint velocities | **RAD/S** | **DEG/S** | 57x scale mismatch |
| EE position | METERS | METERS | OK |
| EE euler angles | RADIANS | RADIANS | OK |

### Why Shaking Occurred

The actor received joint positions like `90.0` (degrees) when it expected `1.57` (radians).

With values 57x larger than expected:
- Network activations saturate
- Output actions become effectively random
- Robot shakes erratically

## Correct Fix

### 1. Change normalization to IDENTITY for STATE

The pretrained network expects raw values, not normalized ones. Update `normalization_mapping`:

```python
normalization_mapping: dict[str, NormalizationMode] = field(
    default_factory=lambda: {
        "VISUAL": NormalizationMode.IDENTITY,  # Encoder handles image normalization
        "STATE": NormalizationMode.IDENTITY,   # RoboBase uses no normalization!
        "ENV": NormalizationMode.IDENTITY,
        "ACTION": NormalizationMode.IDENTITY,  # Actions already in [-1, 1]
    }
)
```

### 2. Convert degrees to radians in observation pipeline

Add conversion in the gym wrapper or buffer BEFORE feeding to actor:

```python
def prepare_state_for_actor(real_state):
    """Convert real robot state (degrees) to Genesis format (radians)."""
    joint_pos = real_state[:6]
    joint_vel = real_state[6:12]
    ee_pos = real_state[12:15]  # Already meters
    ee_euler = real_state[15:18]  # Already radians

    return np.concatenate([
        np.radians(joint_pos),   # degrees -> radians
        np.radians(joint_vel),   # deg/s -> rad/s
        ee_pos,
        ee_euler,
    ])
```

### 3. Update dataset_stats to match Genesis ranges (radians)

```json
"dataset_stats": {
    "observation.state": {
        "min": [-3.14, -3.14, -3.14, -3.14, -3.14, 0.0, -5.0, -5.0, -5.0, -5.0, -5.0, -5.0, 0.0, -0.3, 0.0, -3.14, -3.14, -3.14],
        "max": [3.14, 3.14, 3.14, 3.14, 3.14, 1.0, 5.0, 5.0, 5.0, 5.0, 5.0, 5.0, 0.5, 0.3, 0.4, 3.14, 3.14, 3.14]
    }
}
```

## Revert Incorrect Changes

The normalize_inputs changes added earlier are **incorrect** for RoboBase-trained checkpoints. They should be:
1. Kept for SAC (which uses normalization)
2. Bypassed or set to IDENTITY for DrQ-v2 with RoboBase pretrained weights

## Implementation Options

### Option A: Convert in gym wrapper (preferred)
Add degree-to-radian conversion in `FullProprioceptionWrapper` or `RobotEnv._get_obs()`:
```python
# In observation pipeline
joint_pos_rad = np.radians(joint_pos_deg)
joint_vel_rad = np.radians(joint_vel_deg)
```

### Option B: Convert in buffer conversion
Modify `ReplayBuffer.from_lerobot_dataset()` to convert when loading offline data.

### Option C: Convert in policy select_action
Add conversion before actor inference (but this only fixes inference, not training).

## Files Modified

1. `lerobot/src/lerobot/policies/drqv2/configuration_drqv2.py`
   - Changed default normalization_mapping to IDENTITY for STATE, ENV, ACTION

2. `lerobot/src/lerobot/scripts/rl/gym_manipulator.py`
   - Added `use_radians` parameter to `FullProprioceptionWrapper.__init__`
   - Added degree-to-radian conversion in `observation()` method

3. `lerobot/src/lerobot/utils/buffer.py`
   - Added `convert_to_radians` parameter to `from_lerobot_dataset()`
   - Modified full proprioception computation to output radians when requested

4. `lerobot/src/lerobot/scripts/rl/learner.py`
   - Added `convert_to_radians` extraction from wrapper config
   - Pass `convert_to_radians` to offline buffer loading

5. `outputs/hilserl_drqv2/train_config.json`
   - Added `"use_radians": true` to wrapper config
   - Updated normalization_mapping to all IDENTITY
   - Updated dataset_stats to radian-based ranges

## IMPORTANT: Delete Cached Buffer

After applying this fix, you MUST delete the cached offline buffer to force rebuild with radians:

```bash
rm outputs/hilserl_drqv2/offline_buffer.pt
```

The old cache contains states in degrees, which will cause the same shaking issue.

## Key Insight

**RoboBase does NOT normalize low_dim_state.** The actor network was trained on raw radian values. Our MIN_MAX normalization was converting degree values to [-1, 1], which is completely wrong.

The fix is:
1. Convert degrees → radians (to match Genesis units)
2. Use IDENTITY normalization (to match RoboBase's no-normalization)

## Related

- Devlog 044: HIL-SERL Training Fixes (exploration noise, zero actions)
- Devlog 045: HIL-SERL Online Intervention Fix (MuJoCo FK for interventions)
- `tmp_sim_normalization_inquiry.md`: Inquiry that revealed this issue
