# Devlog 014: IK Orientation Control

## Goal

Enable the IK controller to achieve proper grasp orientation for the SO-101 arm, so fingers can reach a cube on the ground (Z=0.015) without manual joint overrides.

## Problem

The original IK controller only controlled end-effector position. To grasp a cube on the ground, the gripper needs to:
1. Point downward (wrist_flex ~1.65 rad)
2. Have fingers horizontal (wrist_roll = π/2)

Without orientation control, the IK would solve for position but leave the orientation arbitrary, resulting in fingers too high above the cube.

## Solution

### 1. Added Orientation Control to IK Controller

Extended `controllers/ik_controller.py` with:

**Quaternion to Rotation Matrix Conversion:**
```python
def _quat_to_mat(self, quat: np.ndarray) -> np.ndarray:
    """Convert quaternion (w, x, y, z) to 3x3 rotation matrix."""
    w, x, y, z = quat
    return np.array([
        [1 - 2*(y*y + z*z), 2*(x*y - w*z), 2*(x*z + w*y)],
        [2*(x*y + w*z), 1 - 2*(x*x + z*z), 2*(y*z - w*x)],
        [2*(x*z - w*y), 2*(y*z + w*x), 1 - 2*(x*x + y*y)]
    ])
```

**Orientation Error Calculation (Axis-Angle):**
```python
def _orientation_error(self, target_mat: np.ndarray, current_mat: np.ndarray) -> np.ndarray:
    R_error = target_mat @ current_mat.T
    trace = np.trace(R_error)
    cos_theta = np.clip((trace - 1) / 2, -1, 1)
    theta = np.arccos(cos_theta)
    if theta < 1e-6:
        return np.zeros(3)
    axis = np.array([
        R_error[2, 1] - R_error[1, 2],
        R_error[0, 2] - R_error[2, 0],
        R_error[1, 0] - R_error[0, 1]
    ]) / (2 * np.sin(theta))
    return axis * theta
```

**Combined Position + Orientation IK:**
- Stack position Jacobian (3×n) with rotation Jacobian (3×n) → (6×n)
- Stack position error (3,) with orientation error (3,) → (6,)
- Solve combined system with damped least-squares

### 2. Added Locked Joints Feature

New `locked_joints` parameter excludes specific joints from IK solution:

```python
ctrl = ik.step_toward_target(
    target_pos,
    gripper_action=1.0,
    gain=0.5,
    target_quat=target_quat,
    orientation_weight=0.5,
    locked_joints=[3, 4]  # Lock wrist_flex and wrist_roll
)
```

This allows IK to control base/shoulder/elbow while manually controlling wrist orientation.

### 3. Test Scripts

- `test_ik_grasp.py`: Interactive viewer with orientation control
- `test_ik_grasp_headless.py`: Fast headless version
- `test_ik_grasp_video.py`: Saves video to `runs/ik_grasp_test/` for debugging

## Results

With orientation control:
- Both fingers make contact with cube (geoms 27, 28, 29, 30)
- Gripper approaches from correct angle

## Remaining Issues

1. **Finger clipping**: Fingers clip through cube and launch it
   - Partially mitigated with gradual gripper close (ramp over 600 steps)
   - Still occurs occasionally depending on initial conditions

2. **Fingers still too high**: With pure IK orientation control, fingers reach ~Z=0.08 vs cube at Z=0.015
   - The locked_joints + manual override approach works better
   - May need to adjust target_quat or use finger midpoint targeting

## Update: Gradual Gripper Close

Added gradual gripper ramp to reduce clipping:

```python
for step in range(800):
    t = min(step / 600, 1.0)  # Slower gripper close
    gripper_action = 1.0 - 2.0 * t  # Ramp from 1.0 to -1.0
    ctrl = ik.step_toward_target(tcp_target, gripper_action=gripper_action, ...)
```

Results:
- Both fingers make contact (geoms 27, 28, 29, 30)
- Cube lifts slightly (Z: 0.015 → ~0.019)
- Success depends on RNG - sometimes lifts, sometimes clips
- Clipping still occurs but less violently than instant close

## Update: Contact-Based Grasp Detection

**Problem**: Cube was being dropped during lift because the gripper kept closing after achieving grasp, eventually squeezing the cube out before lift started.

**Root cause**: Gripper ramped from 1.0 to -1.0 over 600 steps. Grasp achieved around step 350-400 (contacts on both fingers), but by step 700 gripper over-closed to -0.17 and lost grip.

**Fix**: Stop closing gripper once both fingers contact cube, then maintain that grip force during lift.

```python
# Step 3: Close until grasp achieved
grasp_gripper_action = 1.0
for step in range(800):
    contacts = get_contacts()
    has_27_28 = 27 in contacts or 28 in contacts
    has_29_30 = 29 in contacts or 30 in contacts

    if has_27_28 and has_29_30:
        # Both fingers contacting - lock gripper and start lifting
        break

    t = min(step / 600, 1.0)
    grasp_gripper_action = 1.0 - 2.0 * t
    # ... rest of control loop

# Step 4: Lift with maintained grip
for step in range(300):
    t = min(step / 200, 1.0)
    current_z = lift_start_z + (lift_target_z - lift_start_z) * t
    # Maintain gripper slightly tighter than grasp point
    ctrl = ik.step_toward_target(..., gripper_action=grasp_gripper_action - 0.2, ...)
```

**Results**:
```
--- Step 3: Close ---
  step 352: GRASP ACHIEVED, starting lift

--- Step 4: Lift ---
  step 0: target_z=0.015, cube_z=0.021, contacts=[0, 27, 28, 29, 30]
  step 150: target_z=0.079, cube_z=0.030, contacts=[27, 28, 29, 30]
  step 200: target_z=0.100, cube_z=0.052, contacts=[27, 28, 29, 30]

=== Result ===
Cube Z: 0.0522
Grasping (both fingers): True
Lifted: True
```

Cube successfully lifted to Z=0.052 while maintaining grasp throughout.

## Joint Reference

| Index | Joint | Function |
|-------|-------|----------|
| 0 | base | Base rotation |
| 1 | shoulder | Shoulder pitch |
| 2 | elbow | Elbow pitch |
| 3 | wrist_flex | Wrist pitch (1.65 = point down) |
| 4 | wrist_roll | Wrist roll (π/2 = horizontal fingers) |
| 5 | gripper | Finger open/close |

## Collision Geom Reference

| Geom ID | Part |
|---------|------|
| 27-28 | Gripper/static finger |
| 29-30 | Moving jaw |
| 31 | Cube |
