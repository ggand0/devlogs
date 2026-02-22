# Devlog 024: Fix main_mlp LayerNorm Architecture Mismatch

## Problem

DrQ-v2 policy outputs were saturated at ±1, causing erratic robot behavior:

```
action_logits (pre-tanh): [-14.74, 9.10, 14.82, -5.19]
action (post-tanh): [-1.0, 1.0, 1.0, -0.99]
```

This persisted even after applying fixes from devlog 023 (wrist_roll flip, gripper conversion, MuJoCo FK).

## Root Cause

The `DrQV2Actor` class in `scripts/eval_drqv2.py` incorrectly reconstructed the main_mlp with **LayerNorm** layers that don't exist in the trained checkpoint.

### Original Training Architecture

The DrQ-v2 training code in `robobase` uses `norm="identity"` by default:

```python
# external/robobase/robobase/models/fully_connected.py:166
norm: str = "identity",  # DEFAULT
```

This means the main_mlp structure is:
```
[0] Linear(100, 256)    ← has params
[1] Identity(256)       ← no params (pass-through)
[2] ReLU()              ← no params
[3] Linear(256, 256)    ← has params
[4] Identity(256)       ← no params
[5] ReLU()              ← no params
```

State dict only contains keys at indices 0 and 3 (the Linear layers):
```
actor.main_mlp.0.weight: (256, 100)
actor.main_mlp.0.bias: (256,)
actor.main_mlp.3.weight: (256, 256)
actor.main_mlp.3.bias: (256,)
```

### Buggy Reconstruction

The eval script created:
```python
self.main_mlp = torch.nn.Sequential(
    Linear,      # [0] - loaded ✓
    ReLU,        # [1]
    LayerNorm,   # [2] - NOT IN CHECKPOINT! Random init!
    Linear,      # [3] - loaded ✓
    ReLU,        # [4]
    LayerNorm,   # [5] - NOT IN CHECKPOINT! Random init!
)
```

The spurious LayerNorm layers were initialized with default values (weight=1, bias=0), corrupting the activation distribution and causing extreme action logits.

## Fix

Changed `scripts/eval_drqv2.py` to use `nn.Identity()` instead of `nn.LayerNorm()`:

```python
# Main MLP
# NOTE: Original DrQ-v2 uses norm="identity" (no LayerNorm) by default!
# Structure: [Linear, Identity, ReLU, Linear, Identity, ReLU]
# State dict only has params at indices 0 and 3 (the Linear layers)
self.main_mlp = torch.nn.Sequential(
    torch.nn.Linear(...),
    torch.nn.Identity(),  # norm_fn with norm="identity"
    torch.nn.ReLU(),
    torch.nn.Linear(...),
    torch.nn.Identity(),  # norm_fn with norm="identity"
    torch.nn.ReLU(),
)
```

## Verification

Confirmed no LayerNorm params exist in checkpoint:
```
main_mlp.1: weight=False, bias=False
main_mlp.2: weight=False, bias=False
main_mlp.4: weight=False, bias=False
main_mlp.5: weight=False, bias=False
```

## Key Lesson

When reconstructing a model from a checkpoint, verify the **exact architecture** matches the training code. Default settings (like `norm="identity"`) may differ from common assumptions (like using LayerNorm).

## Files Changed

- `scripts/eval_drqv2.py` - Fixed main_mlp architecture

## Result After Fix

Action logits now in reasonable range:
```
action_logits (pre-tanh): [-1.11, -0.01, 2.91, -1.51]
action (post-tanh): [-0.81, -0.01, 0.99, -0.91]
```

However, robot still exhibits problematic behavior ("rotating right, hitting ground").

---

## Remaining Issues (Discovered During Testing)

### 1. Gripper Action Range Mismatch

**Evidence:**
```python
# Robot expects: [0, 2] where 1 = no-op
# Policy outputs: [-1, 1] (tanh)
# After eval scale (0.05): -0.91 * 0.05 = -0.045

gripper_delta = (action[-1] - 1) * max_gripper_pos
             = (-0.045 - 1) * 50 = -52.27  # Massive closing force!
```

**Location:** `SO101FollowerEndEffector.send_action()` lines 169-175

### 2. Double Action Scaling

**Evidence:**
```
Sim: delta_xyz = action * 0.02 → 0.99 * 0.02 = 19.8mm per step
Real: action * eval_scale * step_sizes = 0.99 * 0.05 * 0.02 = 0.99mm per step
```

The real robot moves **20x slower** than sim due to compounding scales.

### 3. Starting Position Mismatch

**Evidence:**
```python
# Sim training (lift_cube_cartesian.py:207-208):
cube_x = 0.40 + uniform(-0.03, 0.03)  # ~0.40m
cube_y = -0.10 + uniform(-0.03, 0.03)  # ~-0.10m

# Eval script (eval_drqv2.py args):
cube_x = 0.25  # 15cm closer
cube_y = 0.0   # 10cm to the right
```

The policy may not recognize the scene configuration because the workspace is significantly different.

---

## Fix 2: Bypass Broken Gym Action Pipeline

### Problem

Even with tiny action_scale (0.05), robot still made sudden large movements and hit the ground. The gym env's `SO101FollowerEndEffector.send_action()` uses **cached internal state** that diverges from reality:

```python
# SO101FollowerEndEffector.send_action() - BROKEN
if self.current_joint_pos is None:
    self.current_joint_pos = ...  # READ ONCE from robot

# Uses CACHED state for IK
target_joints = self.kinematics.inverse_kinematics(self.current_joint_pos, ...)

# Updates cache from IK OUTPUT, not actual robot!
self.current_ee_pos = desired_ee_pos.copy()  # DESIRED, not ACTUAL
self.current_joint_pos = target_joint_values_in_degrees.copy()  # IK OUTPUT, not ACTUAL
```

After step 1, `self.current_joint_pos` is whatever IK computed, **not what the robot actually reached**. Internal state immediately diverges from reality.

### Solution

Bypassed the gym env's action pipeline entirely. Use **direct MuJoCo IK** like `rl_inference.py` does:

```python
# Read ACTUAL joint positions from robot EVERY step
current_joints_rad = get_joint_positions_rad(robot)

# Use MuJoCo IK (same as sim training)
target_joints_rad = ik.cartesian_to_joints(
    delta_xyz,
    current_joints_rad,
    action_scale=args.action_scale,
    locked_joints=[4],  # Lock wrist_roll for stability
)

# Send joint targets directly to robot
send_joint_action_degrees(robot, target_joints_rad, gripper_action)
```

Key differences:
1. **Read actual robot state every step** (not cached internal state)
2. **Use MuJoCo IK** (same as sim training, not placo)
3. **Direct joint control** (bypass broken EE control path)

### Also Fixed

- Changed default `action_scale` from 1.0 to 0.02 (2cm per action unit, same as sim)
- Use MuJoCo FK for observations (to match sim training)
- Added control rate timing (10 Hz)

### Result

**Fix confirmed working.** Robot now exhibits same smooth motion as `rl_inference.py` - no more sudden movements or hitting ground.

Remaining issue: Policy doesn't grab cube (finger flapping behavior). This is a sim-to-real gap issue with the policy itself, not an action execution bug.

---

## Fix 3: Upstream Lerobot with MuJoCo IK

### Problem

The `eval_drqv2.py` fix (bypassing gym action pipeline) works, but HIL-SERL training uses `train_hilserl` which still goes through the broken `SO101FollowerEndEffector.send_action()` path in lerobot.

### Solution

Rewrote `SO101FollowerEndEffector` in lerobot to use MuJoCo IK directly instead of placo:

1. **Config changes** (`config_so101_follower.py`):
   - Replaced `urdf_path` with `mujoco_model_path`
   - Replaced `target_frame_name` with `end_effector_site`
   - Added `ik_damping`, `ik_max_dq` parameters
   - Added `locked_joints` config (default: `[4]` to lock wrist_roll)
   - Added `action_scale` (default: 0.02m per action unit)
   - Removed `end_effector_step_sizes` (redundant with action_scale)

2. **Robot class changes** (`so101_follower_end_effector.py`):
   - Load MuJoCo model in `__init__`
   - Added `_sync_mujoco()` for FK
   - Added `_compute_ik()` with damped least-squares
   - `send_action()` now:
     - Reads actual joint positions every step (no caching)
     - Uses MuJoCo FK to get current EE position
     - Uses MuJoCo IK to compute target joints
   - Removed broken state caching logic

### Key Code

```python
# ALWAYS read current joint positions from robot (not cached)
current_pos_dict = self.bus.sync_read("Present_Position")
current_joints_deg = np.array([current_pos_dict[name] for name in self.JOINT_NAMES])
current_joints_rad = np.deg2rad(current_joints_deg)

# Get current EE position from MuJoCo FK
self._sync_mujoco(current_joints_rad)
current_ee_pos = self._get_ee_position()

# Compute target EE position
delta_xyz = action[:3] * self.config.action_scale
target_ee_pos = current_ee_pos + delta_xyz

# Compute IK to get target joint positions (radians)
target_joints_rad = self._compute_ik(target_ee_pos, current_joints_rad)
```

### Files Changed

- `lerobot/src/lerobot/robots/so101_follower/config_so101_follower.py`
- `lerobot/src/lerobot/robots/so101_follower/so101_follower_end_effector.py`

### Usage

```python
from lerobot.robots.so101_follower import SO101FollowerEndEffector, SO101FollowerEndEffectorConfig

config = SO101FollowerEndEffectorConfig(
    port="/dev/ttyACM0",
    mujoco_model_path="/path/to/lift_cube.xml",
    end_effector_site="gripperframe",
    action_scale=0.02,  # 2cm per action unit
    locked_joints=[4],  # Lock wrist_roll
)
robot = SO101FollowerEndEffector(config)
```

---

## Related

- Devlog 023: DrQ-v2 Action Saturation Debug (prior debugging)
- `/home/gota/ggando/ml/pick-101/external/robobase/robobase/models/fully_connected.py` - Source of truth for architecture
- `scripts/rl_inference.py` - Reference implementation that works correctly
