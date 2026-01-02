# Devlog 007: Gripper Clipping and Physics Fix

## Issue Discovery

After V1-V4 reward iterations, the agent consistently found exploits that avoided actual lifting. Closer inspection with a side-view camera revealed the root cause: **the gripper tip was clipping through the cube geometry**.

### Symptoms
- Cube z-position dropping to 0.007 (below baseline 0.01)
- Agent getting stable ~180 reward without lifting
- `contacts=(True, False)` - only static gripper touching, not pinching

### Video Evidence
Added multi-camera eval output to diagnose:
- `eval_*_closeup.mp4` - side view at cube level
- `eval_*_wide.mp4` - diagonal overview (azimuth=135)
- `eval_*_wide2.mp4` - diagonal from other side (azimuth=45)

The closeup view clearly showed the gripper tip penetrating into the cube, pushing it into the ground.

## Root Cause

Two compounding issues:

### 1. Mesh Collision Gaps
The gripper uses STL mesh files for collision detection. Thin parts of the mesh (like the gripper tip) don't have reliable collision - MuJoCo's mesh collision can miss contacts on thin geometry.

### 2. Soft Default Contact Parameters
MuJoCo's default contact parameters are quite soft:
- `solref="0.02 1"` - 20ms time constant (soft spring)
- `solimp="0.9 0.95 0.001"` - allows 5-10% penetration

This is equivalent to **soft rubber or foam** (~1-10 MPa), not hard plastic.

## Fixes Applied

### Fix 1: Added Box Collision Primitives to Gripper Tips

In `SO-ARM100/Simulation/SO101/so101_new_calib.xml`:

```xml
<!-- Static gripper tip -->
<geom name="static_gripper_tip" type="box" class="collision"
      size="0.008 0.006 0.018" pos="-0.008 0 -0.088" rgba="1 0 0 0.3"/>

<!-- Moving jaw tip -->
<geom name="moving_jaw_tip" type="box" class="collision"
      size="0.006 0.004 0.015" pos="0 -0.055 0.019" rgba="0 1 0 0.3"/>
```

Box primitives have reliable collision detection unlike mesh geometry.

### Fix 2: Stiffer Contact Parameters

In `SO-ARM100/Simulation/SO101/lift_cube_scene.xml`:

```xml
<option timestep="0.002" gravity="0 0 -9.81" noslip_iterations="3"/>

<default>
    <geom solref="0.001 1" solimp="0.99 0.99 0.001"/>
</default>
```

| Parameter | Before | After | Real-world equivalent |
|-----------|--------|-------|----------------------|
| solref timeconst | 0.02s | 0.001s | Soft rubber → Hard plastic |
| solimp penetration | 5-10% | ~1% | Foam → Wood/plastic |

The new settings are more realistic for PLA plastic gripper + wooden cube (planned for IRL experiments).

## Results

### Before Fix
```
cube_z=0.007  # Pushed below ground baseline
contacts=(True, False)  # Clipping, not grasping
```

### After Fix
```
cube_z=0.010-0.013  # Stays at or above baseline
contacts=(True, True)  # Proper contact detection
```

The physics exploit is closed. The existing trained model no longer works well (since it learned to exploit the soft physics), but new training should learn proper grasping behavior.

## Files Modified

- `SO-ARM100/Simulation/SO101/so101_new_calib.xml` - Added gripper tip collision boxes
- `SO-ARM100/Simulation/SO101/lift_cube_scene.xml` - Stiffer contact parameters
- `envs/lift_cube_cartesian.py` - Multi-camera render support
- `eval_cartesian.py` - Export closeup/wide/wide2 videos

## Post-Fix Training Results (FAILED)

After implementing the collision box fix, we retrained with V3 and V4 rewards. Both failed:

- **V4 (500k)**: Model regressed, staying far from cube with open gripper
- **V3 (500k)**: Same regression pattern - promising at 250k, worse at 500k

See [devlog 008](008_post_clipping_fix_training.md) for detailed results.

## Bug Discovery: Collision Boxes Blocked Grasping

Investigation revealed TWO compounding bugs that made the "fix" ineffective:

### Bug 1: Geom ID Shift

When we added the box collision primitives (`static_gripper_tip` and `moving_jaw_tip`), the geom IDs in the XML shifted. The contact detection code used **hardcoded geom ID ranges**:

```python
# Original hardcoded ranges (WRONG after adding boxes)
gripper_geom_ids = set(range(25, 29))  # Expected gripper mesh geoms
jaw_geom_ids = set(range(29, 31))       # Expected moving jaw mesh geoms
```

After adding boxes:
- `static_gripper_tip` = geom ID 29
- `moving_jaw_tip` = geom ID 32
- Gripper mesh geoms shifted to 25-28 (unchanged)
- Moving jaw mesh geoms shifted to 30-31

The contact detection was checking the **wrong geom IDs**, so grasp detection never triggered.

### Bug 2: Collision Boxes Physically Blocked Grasping

Visual inspection with collision geoms enabled (group 3) revealed the boxes were:
- **Protruding inward** into the gripper opening
- **Blocking the cube** from fitting between the gripper jaws

The collision boxes meant to help with contact detection were actually **preventing physical grasping**.

## Solution: Revert Boxes, Add Push-Down Penalty (V5)

Since the physics approach failed due to collision geometry complexity, we're reverting to reward-based correction:

### Changes

1. **Removed collision boxes** from `so101_new_calib.xml`
   - Removed `static_gripper_tip` box
   - Removed `moving_jaw_tip` box

2. **Reverted contact detection** to hardcoded geom ID ranges
   - Back to original `range(25, 29)` for gripper geoms
   - Back to original `range(29, 31)` for jaw geoms

3. **Added V5 push-down penalty** in reward function:
   ```python
   # V5: Push-down penalty - penalize cube below baseline (z=0.01)
   if cube_z < 0.01:
       push_penalty = (0.01 - cube_z) * 50.0
       reward -= push_penalty
   ```

### V5 Reward Structure

```python
reward = 0.0

# Reach reward
reward += 1.0 - np.tanh(10.0 * gripper_to_cube)

# Push-down penalty (NEW in V5)
if cube_z < 0.01:
    reward -= (0.01 - cube_z) * 50.0

# Lift baseline
reward += max(0, (cube_z - 0.01) * 10.0)

# Grasp bonus + lift multiplier
if is_grasping:
    reward += 0.25
    reward += max(0, (cube_z - 0.01) * 40.0)

# Binary lift bonus
if cube_z > 0.02:
    reward += 1.0

# Success bonus
if is_success:
    reward += 10.0
```

### Reward at Different Heights (V5)

| cube_z | No Grasp | With Grasp | Notes |
|--------|----------|------------|-------|
| 0.007 | 0.75 (reach) - 0.15 (penalty) = 0.60 | 0.60 + 0.25 = 0.85 | Push-down penalized |
| 0.010 | 0.90 (reach) | 0.90 + 0.25 = 1.15 | Baseline (no penalty) |
| 0.015 | 0.90 + 0.05 = 0.95 | 0.95 + 0.25 + 0.20 = 1.40 | Small lift |
| 0.020 | 0.90 + 0.10 + 1.0 = 2.00 | 2.25 + 0.40 = 2.65 | Binary threshold |
| 0.080 | 0.90 + 0.70 + 1.0 = 2.60 | 2.85 + 2.80 = 5.65 | Success height |

Key difference from V3: Pushing cube to z=0.007 now gives **lower** reward than baseline.

## Next Steps

1. Train V5 fresh from scratch
2. Monitor cube_z to verify push-down penalty works
3. If still regressing, consider curriculum learning (start with elevated cube)

## Files Modified

- `SO-ARM100/Simulation/SO101/so101_new_calib.xml` - Removed collision boxes
- `envs/lift_cube.py` - Added V5 push-down penalty, reverted contact detection
- `envs/lift_cube_cartesian.py` - Reverted contact detection
