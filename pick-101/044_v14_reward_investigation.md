# V14 Reward Investigation: Single-Finger Behavior Analysis

## Summary

v14 reward function was introduced to fix the 7cm plateau issue from v13. However, training with v14 showed persistent single-finger behavior at 1.12M steps. Investigation revealed two key findings:

1. **Action penalty bug**: Original v14 had `hold_count > 0` condition that never triggered for failed grasps
2. **Camera change**: Wrist camera was significantly recalibrated between v13 success and v14 training

## V13 to V14 Changes

### V13 Reward (commit 143f840)
```python
# Action rate penalty for smoothness (only when lifted, to not hinder lifting)
if action is not None and cube_z > 0.06:
    action_delta = action - self._prev_action
    action_penalty = 0.01 * np.sum(action_delta**2)
    reward -= action_penalty
```

### V14 Original (commit 64672e4)
```python
# Action rate penalty ONLY during hold phase (not before reaching target)
if action is not None and hold_count > 0:
    action_delta = action - self._prev_action
    action_penalty = 0.02 * np.sum(action_delta**2)
    reward -= action_penalty
```

### V14 Fixed (commit 314291b)
```python
# Action rate penalty ONLY during hold phase at target height
if action is not None and cube_z > self.lift_height and hold_count > 0:
    action_delta = action - self._prev_action
    action_penalty = 0.02 * np.sum(action_delta**2)
    reward -= action_penalty
```

## Action Penalty Bug Analysis

The original v14 bug: `hold_count > 0` without height check.

**Why it matters:**
- `hold_count` only increments when `is_lifted = is_grasping AND cube_z > lift_height`
- If agent never properly grasps, `hold_count` stays 0 forever
- Result: No action penalty ever applies for single-finger motion

**Why the fix is still problematic:**
- Fixed to: `cube_z > lift_height AND hold_count > 0`
- But single-finger behavior happens at **ground level** during grasp phase
- Action penalty only applies when cube is **lifted** (z > 0.08m)
- Therefore: Action penalty timing is **irrelevant** to ground-level single-finger behavior

## Camera Change Between V13 and V14

Major discovery: The wrist camera was significantly recalibrated between v13 training success and current v14 training.

### V13 Camera (old)
```xml
<camera name="wrist_cam" pos="0.0 -0.055 0.02" euler="0 0 3.14159" fovy="75"/>
```

### Current Camera (calibrated v3)
```xml
<camera name="wrist_cam" pos="0.02 -0.08 -0.06" euler="0.698 0 3.14159" fovy="103"/>
```

### Camera Parameter Comparison

| Parameter | V13 | Current | Change |
|-----------|-----|---------|--------|
| Position X | 0.0 | 0.02 | +0.02 (offset to center fingers) |
| Position Y | -0.055 | -0.08 | -0.025 (further forward) |
| Position Z | 0.02 | -0.06 | -0.08 (much closer to fingertips) |
| Pitch | 0° | +40° | Tilted down toward workspace |
| FOV | 75° | 103° | Much wider field of view |

**Impact**: The agent sees a completely different image. V13 success was trained on the old camera. Current v14 training uses the new calibrated camera - essentially a different visual learning problem.

## Single-Finger Behavior Root Cause

The single-finger behavior is NOT caused by the action penalty timing change. At ground level, v13 and v14 have **identical reward structures**:

| Component | V13 | V14 |
|-----------|-----|-----|
| Reach reward | `1.0 - tanh(10 * dist)` | Same |
| Grasp bonus | +0.25 if is_grasping | Same |
| Push-down penalty | if cube_z < 0.01 | Same |
| Drop penalty | -2.0 if was_grasping | Same |
| Lift rewards | Gated on is_grasping | Same |

**The fundamental issue**: Reach reward (~1.0) dominates grasp bonus (+0.25). Agent can achieve high reward by positioning close to cube without proper two-finger grasp.

### Potential Solutions (not yet implemented)

1. **Increase grasp bonus**: +1.0 instead of +0.25
2. **Add contact-based shaping**: Reward two-finger contact explicitly
3. **Penalize single-finger contact**: Asymmetric contact penalty
4. **Reduce reach reward**: Scale down when not grasping

## Timeline of Changes

| Commit | Description |
|--------|-------------|
| 143f840 | Add v13 reward (old camera, worked at 800k steps) |
| fbc4ef3 | Update wrist camera to calibrated v3 settings |
| 2f73ae9 | Apply calibrated v3 camera, update config to v14 |
| 64672e4 | Add v14 reward (action penalty bug) |
| 314291b | Fix v14 action penalty (add height check) |

## Conclusion

The comparison "v13 worked but v14 doesn't" conflates two changes:
1. Reward function: v13 → v14 (action penalty timing)
2. Camera: old → calibrated v3 (completely different visual input)

The v13 success was achieved with the **old camera**. Current v14 training runs with the **new camera**. To properly compare reward functions, would need to either:
- Train v14 with old camera settings
- Train v13 with new camera settings
- Accept that new camera requires more training time to converge

**Current status**: Training continues with v14 + new camera. Single-finger behavior may resolve with more training steps, or may require reward function adjustments.
