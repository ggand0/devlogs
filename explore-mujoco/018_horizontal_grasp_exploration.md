# Horizontal Gripper IK Reset Exploration

## Goal

Simplify the IK-based reset by rotating the wrist 90 degrees so the gripper is horizontal, potentially allowing pure IK without locked wrist joints.

Current reset (in `lift_cube.py`) uses:
- `wrist_flex=1.65` (pointing down)
- `wrist_roll=pi/2` (horizontal fingers)
- `locked_joints=[3, 4]` to prevent IK from changing wrist angles

The hypothesis was that a horizontal gripper configuration could use pure IK (no locked joints) for a simpler implementation.

## Experiments

### Test 1: Pure IK with Horizontal Gripper

Script: `tests/test_horizontal_grasp.py`

Configuration:
- `wrist_roll=pi/2` (horizontal fingers)
- Target graspframe (between fingers) directly
- No locked joints - let IK solve freely

Result at cube Z=0.015 (table level):
```
Graspframe: [0.253, 0.0, 0.053]
Target:     [0.25, 0.0, 0.015]
Error:      0.038
```

IK converges to Z=0.053 but cannot reach Z=0.015. The arm geometry prevents reaching table level with horizontal gripper.

### Test 2: Different Initial Configurations

Tried initializing with elbow "leaned back" (`elbow_flex=-1.0`) to help IK find a better solution.

Result: IK moved elbow from -1.0 to +0.894 (forward), finding a local minimum that still couldn't reach low.

### Test 3: Locked Elbow

Locked `elbow_flex=-1.2` to force the "lean back" configuration.

Result: IK compensated by rotating `wrist_flex` to 1.659 (pointing down), essentially recreating the top-down approach.

### Test 4: Multiple Cube Heights (Initial - Bug)

Initial test spawned cubes mid-air, but they fell to table before gripper reached them. IK was targeting spawn position, not actual position.

### Test 5: Fixed - Target Actual Cube Position

After letting cube settle and targeting actual position:

| Spawn Z | Settled Z | IK Error | wrist_flex | Result |
|---------|-----------|----------|------------|--------|
| 0.015   | 0.015     | 0.038    | -0.02      | FAIL   |
| 0.030   | 0.015     | 0.038    | -0.02      | FAIL   |
| 0.050   | 0.015     | 0.038    | -0.02      | FAIL   |
| 0.070   | 0.015     | 0.038    | -0.02      | FAIL   |
| 0.090   | 0.015     | 0.038    | -0.02      | FAIL   |

All cubes fall to table (Z=0.015). IK consistently reaches Z=0.053 but cannot go lower - **geometric limitation confirmed**.

## Findings

1. **Geometric limitation**: The SO-101 arm cannot reach table level (Z=0.015) with a truly horizontal gripper. Minimum reachable Z with horizontal orientation is ~0.05.

2. **IK compensation**: When forced to reach low, IK rotates wrist_flex downward, effectively recreating the top-down approach.

3. **Grasp stability**: Even when IK reaches the target (at higher Z), the horizontal approach doesn't create a stable grasp. The cube is contacted but not securely pinched, causing it to fall during lift.

4. **Why top-down works**: The top-down approach succeeds because:
   - Gripper descends from above with fingers straddling the cube
   - Fingers close around cube sides, creating secure pinch
   - Wrist angle allows reaching table level

## Joint Configuration Comparison

**Top-down (working)**:
```
wrist_flex: 1.65  (pointing down)
wrist_roll: 1.57  (pi/2, horizontal fingers)
```

**Horizontal (not working)**:
```
wrist_flex: ~0    (horizontal)
wrist_roll: 1.57  (pi/2, horizontal fingers)
```

## Conclusion

The horizontal gripper approach is geometrically limited for table-level grasping with the SO-101 arm. The current top-down reset with locked wrist joints remains the correct approach.

A horizontal approach could work if:
1. The cube is on a raised platform (Z >= 0.07)
2. Different gripper geometry that can reach lower
3. Different arm configuration

## Files Created

- `tests/test_horizontal_grasp.py` - Tests horizontal gripper at various heights
- `tests/test_ik_side_grasp.py` - Compares top-down vs side approach configurations

## Next Steps

- Keep current top-down reset implementation
- Consider horizontal approach only if task involves raised objects
- Investigate why grasp detection succeeds but lift fails (grasp stability issue)
