# Devlog 035: Sim Observation Playback Test & Coordinate Frame Discovery

**Date**: 2025-01-12
**Status**: IK Fixed, Forward Motion Working

## Summary

Implemented sim observation playback to diagnose sim2real issues. Discovered and fixed two critical issues:
1. **Coordinate frame mismatch** between Genesis (-Y forward) and MuJoCo (+X forward) - fixed with `--genesis_to_mujoco` transform
2. **IK joint offset mismatch** - IK was computing from wrong position due to sensor offset - fixed by applying `apply_joint_offset()` before IK

Robot now moves forward correctly with `--action_scale 0.05`.

## New Features Added

### `--sim_frames_dir` and `--sim_states_file` options

Added to `ppo_inference.py` to load pre-recorded Genesis observations:

```bash
# Full sim observation playback
uv run python scripts/ppo_inference.py \
    --checkpoint /path/to/checkpoint.pt \
    --sim_frames_dir ./genesis_episode_obs \
    --sim_states_file ./genesis_episode_obs/states.npy \
    --dry_run \
    --episode_length 50
```

### `--genesis_to_mujoco` option

Transforms actions from Genesis coordinate frame to MuJoCo coordinate frame:

```bash
uv run python scripts/ppo_inference.py \
    --checkpoint /path/to/checkpoint.pt \
    --genesis_to_mujoco \
    --debug_state \
    --episode_length 50
```

## Coordinate Frame Mismatch (CONFIRMED)

| Direction | Genesis Frame | MuJoCo Frame |
|-----------|--------------|--------------|
| Forward   | **-Y**       | **+X**       |
| Sideways  | X            | Y            |
| Up/Down   | Z            | Z            |

### Genesis Setup
```python
ROBOT_POS = (0.25, 0.314, 0.0)  # At table edge
ROBOT_EULER = (0, 0, -90)       # Facing -Y
# Cube spawns at (0.25, ~0, z) - in front of robot
# Forward = -Y direction
```

### MuJoCo Setup
```python
# Robot at origin (0, 0, 0)
# Cube at (0.25, 0, z) - forward from robot
# Forward = +X direction
```

### Action Transformation (Implemented)

```python
# Genesis action: (X_sideways, Y_forward, Z_up)
# MuJoCo action:  (X_forward, Y_sideways, Z_up)
if args.genesis_to_mujoco:
    genesis_x, genesis_y, genesis_z = delta_xyz
    delta_xyz = np.array([
        -genesis_y,  # MuJoCo X (forward) = -Genesis Y
        genesis_x,   # MuJoCo Y (sideways) = Genesis X
        genesis_z,   # Z unchanged
    ])
```

## Test Results with `--genesis_to_mujoco`

### Policy Output Analysis

```
[DEBUG] Step 0: raw_action=[ 0.095, -0.269, -0.158, 0.186]
[DEBUG] Transformed: genesis(0.095,-0.269,-0.158) → mujoco(0.269,0.095,-0.158)
```

After transformation:
- **MuJoCo X = 0.27** (forward) ✓ - should move forward
- **MuJoCo Y = 0.09** (sideways) - slight right motion
- **MuJoCo Z = -0.16** (DOWN) - this is why gripper drops!
- **Gripper = 0.19** (slightly open)

### Real Robot Behavior

| Observation | Value | Interpretation |
|-------------|-------|----------------|
| delta X (forward) | +0.27 | Should move forward |
| delta Z (vertical) | -0.16 | Goes DOWN toward table |
| EE position change | None! | IK not executing forward motion |
| Gripper | ~0.2 (open) | Twitches around open state |

**Key Issue**: Despite commanding X=0.27 (forward), the EE position stays at `[0.272, -0.014, 0.030]` and doesn't move forward.

### IK Debug Output

```
Initial EE position: [ 0.27172705 -0.01358983  0.03002386]
Step 0: delta=[0.27,0.09,-0.16] gripper=0.19 ee=[0.272,-0.014,0.030]  <-- NO CHANGE!
Step 10: delta=[0.27,0.10,-0.15] gripper=0.19 ee=[0.272,-0.014,0.013] <-- Only Z changed
```

The Z motion (dropping) is being executed, but the X motion (forward) is NOT.

## Root Cause: IK Joint Offset Mismatch (FIXED)

The IK was computing from the wrong starting position due to sensor offset not being applied.

### Problem
```python
# IK was using RAW sensor joints
ik.sync_joint_positions(current_joints)  # Wrong!
# But display/state used CORRECTED joints
ik.sync_joint_positions(apply_joint_offset(joint_pos_rad))  # Correct
```

The elbow_flex sensor reads ~12.5° more bent than actual physical position. Without correction:
- IK thought EE was at position A
- Actual EE was at position B
- IK computed motion from A, but robot moved from B → wrong direction

### Fix Applied
```python
# Apply joint offset for accurate IK
current_joints_raw = robot.get_joint_positions_radians()
current_joints = apply_joint_offset(current_joints_raw)

target_joints = ik.cartesian_to_joints(delta_xyz, current_joints, ...)

# Convert back to raw joints for robot command
target_joints[2] -= ELBOW_FLEX_OFFSET_RAD
```

### Additional Fix: Lock Both Wrist Joints
Changed from `locked_joints=[4]` to `locked_joints=[3, 4]` to prevent wrist_pitch coupling.

## Successful Test Results

With fixes applied and `--action_scale 0.05`:

```
Step 0:  ee=[0.272,-0.014,0.032]
Step 10: ee=[0.270,-0.010,0.005]  # Z dropped to table level
Step 20: ee=[0.278,-0.010,0.001]  # Moving FORWARD ✓
Step 30: ee=[0.305,-0.008,0.003]  # +33mm forward from start
```

**Forward motion now working!** Policy exhibits "approach cube" behavior:
- X increases (moving forward toward cube)
- Z drops to table level (ready to grasp)
- Gripper ~0.2 (slightly open, approach mode)

## Files Modified

- `scripts/ppo_inference.py`:
  - Added `--sim_frames_dir`, `--sim_states_file` for sim observation playback
  - Added `--genesis_to_mujoco` for coordinate frame transformation
  - Added IK debug output (target_ee, joint_diff)
  - Changed status output to show full EE position `ee=[X,Y,Z]`

## Diagnostic Matrix

| Scenario | Result | Interpretation |
|----------|--------|----------------|
| Without transform | Robot drifts sideways | Coordinate frame mismatch ✓ |
| With `--genesis_to_mujoco` | Robot drops, no forward | Transform working, IK issue |
| IK target vs actual | X unchanged, Z changes | IK can't reach forward target |

## Policy Behavior Summary

The 100k checkpoint policy outputs:
- **Forward motion** (Genesis -Y → MuJoCo +X after transform): ~0.27
- **Down motion** (Z): ~-0.15 (dropping to grasp)
- **Gripper**: ~0.2 (open, slight twitch)

This is reasonable "approach cube" behavior - the policy learned to go forward and down. But:
1. Forward motion isn't being executed by IK
2. Down motion IS being executed (gripper drops to table)

## Next Steps

1. **Test with real camera** - Currently using mock observations; need to verify with real visual input
2. **Verify grasp behavior** - Policy should close gripper when cube is detected
3. **Test with correctly-trained checkpoint** - Current checkpoint trained with wrong wrist config (joint[4]=+π/2)
4. **Record sim observations** - For sim2real visual gap diagnosis

## Commands

```bash
# Working command with all fixes
uv run python scripts/ppo_inference.py \
    --checkpoint /home/gota/ggando/ml/pick-101-genesis/runs/ppo_multi_env_wrist/20260111_234921/checkpoints/checkpoint_100352.pt \
    --genesis_to_mujoco \
    --debug_state \
    --action_scale 0.05 \
    --episode_length 40

# With real camera (when available)
uv run python scripts/ppo_inference.py \
    --checkpoint /path/to/checkpoint.pt \
    --genesis_to_mujoco \
    --camera_index 1 \
    --action_scale 0.05 \
    --episode_length 50
```

## Recording Genesis Episodes

```python
# In Genesis eval code
save_dir = "episode_obs"
os.makedirs(save_dir, exist_ok=True)

frames, states, actions = [], [], []
for step in range(episode_length):
    obs = env.get_image_obs()
    frames.append(obs['rgb'].cpu().numpy())
    states.append(obs['low_dim_state'].cpu().numpy())

    action = policy.get_action(obs)
    actions.append(action.cpu().numpy())
    env.step(action)

np.save(f"{save_dir}/frames.npy", np.array(frames))
np.save(f"{save_dir}/states.npy", np.array(states))
np.save(f"{save_dir}/actions.npy", np.array(actions))
```
