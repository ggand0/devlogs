# IK Joint Movement Fixes for HIL-SERL Training

## Problem

During HIL-SERL training, the robot only moved joint 1 (shoulder_lift) while other joints remained stationary. Joint 1 appeared to "shake" at its position.

## Root Causes Identified

### 1. Reset Position Outside MuJoCo Joint Limits

The `fixed_reset_joint_positions` in the config had values outside MuJoCo's model limits:

| Joint | Reset Position | MuJoCo Limit | Issue |
|-------|---------------|--------------|-------|
| shoulder_lift | -104.84° | ±100° | **Outside limit** |
| elbow_flex | 97.05° | ±96.8° | **Outside limit** |

When IK computed targets for joints starting outside limits, the clipping step forced them back toward limits, preventing meaningful control from policy actions.

**Fix**: Updated reset positions to be within MuJoCo limits:
```json
// Before
"fixed_reset_joint_positions": [-0.4, -104.84, 97.05, 44.09, 89.45, 4.35]

// After
"fixed_reset_joint_positions": [-0.4, -95.0, 90.0, 90.0, 89.45, 4.35]
```

### 2. End-Effector Starting Outside Workspace Bounds

The EE position at reset was outside the configured workspace bounds:

| Axis | EE Position | Bounds Min | Issue |
|------|-------------|------------|-------|
| X | 0.12m | 0.15m | **Outside bound** |
| Z | -0.01m | 0.02m | **Outside bound** |

Every action got clipped to try to reach bounds minimum, causing large IK movements that hit joint limits.

**Fix**: Expanded EE bounds to include actual workspace:
```json
// Before
"end_effector_bounds": {"min": [0.15, -0.15, 0.02], "max": [0.35, 0.15, 0.25]}

// After
"end_effector_bounds": {"min": [0.10, -0.15, -0.02], "max": [0.35, 0.15, 0.25]}
```

### 3. MuJoCo Joint Limits More Restrictive Than Real Robot

The MuJoCo model has joint limits (e.g., shoulder_lift ±100°) that are more restrictive than the real robot's physical limits (~±110°). The IK was clamping targets to MuJoCo limits, artificially restricting the robot's range.

**Fix**: Removed joint limit clamping in IK for real robot operation. The real robot hardware enforces its own physical limits.

```python
# Before: Clamp to MuJoCo limits
target_joints = np.clip(target_joints, self.joint_limits_lower, self.joint_limits_upper)

# After: Let real robot hardware enforce its own limits
# (clamping code removed)
```

## Files Modified

1. **configs/reach_grasp_hilserl_train_config.json**
   - Fixed `fixed_reset_joint_positions` to be within MuJoCo limits
   - Expanded `end_effector_bounds` to include actual workspace

2. **lerobot/robots/so101_follower/so101_follower_end_effector.py**
   - Removed MuJoCo joint limit clamping in `_compute_ik()`
   - Added debug logging for EE bounds clipping
   - Added joint limits logging at initialization

## Debug Logging Added

The following logging was added to help diagnose issues:

1. **Joint limits at init**: Logs MuJoCo joint limits in degrees
2. **EE bounds clipping**: Warns when target EE position is clipped to workspace bounds
3. **SEND_ACTION**: Logs full action flow including:
   - Raw action values
   - Scaled delta_xyz
   - Current/target EE positions
   - Current/target joint positions (degrees)
   - Joint deltas (degrees)

## Verification

After fixes, logs should show:
- No `IK CLIPPING` warnings (joint limit clamping removed)
- Minimal `EE BOUNDS CLIPPING` warnings
- Non-zero `joint_deltas_deg` for all active joints (0, 1, 2)
