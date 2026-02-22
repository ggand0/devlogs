# Devlog 034: Genesis PPO Sim-to-Real Debug

**Date**: 2025-01-11
**Status**: In Progress

## Summary

Debugging sim-to-real transfer for Genesis-trained PPO policy on real SO-101 robot. The policy runs but exhibits two failure modes instead of successful grasping.

## Observed Behaviors

1. **Gripper lowers toward cube, then stops** - Policy moves down to grasp height, starts closing gripper, but doesn't complete the grasp
2. **Drift to the right (negative Y)** - Policy consistently outputs negative Y actions causing lateral drift

## Fixes Applied

### 1. State Dimension Mismatch (Fixed)
- **Issue**: PPO expects 18-dim state, `LowDimStateBuilder` outputs 21-dim (includes 3 zeros for cube_pos)
- **Fix**: Slice state to 18 dims: `low_dim_obs[:18]`
- **Error before fix**: `RuntimeError: mat1 and mat2 shapes cannot be multiplied (1x21 and 18x64)`

### 2. Gripper Reset State (Fixed)
- **Issue**: Real robot reset used `gripper=1.0` (fully open), Genesis uses `gripper=0.3` (partially open)
- **Fix**: Added `RESET_GRIPPER = 0.3` constant and updated reset sequence
- **Location**: `lift_cube_env.py:519` shows `gripper_open = torch.ones(...) * 0.3`

### 3. Reset Position (Previously Fixed)
- Genesis `curriculum_stage=3` starts at **grasp height** (Z=0.02), not above cube (Z=0.05)
- Target: `[0.25, -0.015, 0.02]`

## Debug Options Added

```bash
--save_obs          # Save first 5 observation images as PNG
--save_obs_video    # Save all observations as video
--rotate_image N    # Rotate image by N degrees (0, 90, 180, 270)
--flip_y            # Negate Y-axis of actions
--debug_state       # Print verbose state/action breakdown
--genesis_wrist     # Use Genesis wrist config (joint[4]=+π/2 instead of -π/2)
```

## Investigated But Not the Issue

### FOV Mismatch
- Genesis: 86° vertical FOV
- Real camera: 130° diagonal FOV → ~78° vertical after square crop
- **Conclusion**: Similar enough, not the main issue

### Coordinate System
- Genesis uses Z-up (same as MuJoCo)
- Robot coordinate frame matches
- `--flip_y` made behavior worse (gripper dropped to ground)

### Image Preprocessing
- Both use CHW format, RGB color order
- Center crop to square, resize to 84x84
- Visual comparison shows similar views

### Euler Angle Conversion
- Genesis: quaternion → euler (roll, pitch, yaw)
- Real robot: rotation matrix → euler (roll, pitch, yaw)
- Same convention, should produce matching results

## State Vector Format

```
Genesis low_dim_state (18 dims for PPO):
├── joint_pos (6): 5 arm joints + 1 gripper
├── joint_vel (6): 5 arm joints + 1 gripper
├── gripper_pos (3): EE XYZ position
└── gripper_euler (3): roll, pitch, yaw
```

## Action Output Analysis

Typical policy output:
```
Step 0:  delta=[-0.01, -0.14, 0.07] gripper=-0.30
Step 20: delta=[ 0.00, -0.06, 0.05] gripper=-0.18
Step 40: delta=[ 0.00, -0.06, 0.05] gripper=-0.18
```

Observations:
- **X**: Small negative (moving backward slightly)
- **Y**: Consistently negative (-0.06 to -0.14) - causing right drift
- **Z**: Positive (0.05-0.07) but EE actually lowers - action interpretation may be correct
- **Gripper**: Negative (closing) - correct for grasping

## Root Cause: Wrist Configuration Mismatch

**CONFIRMED**: Genesis trains with wrong wrist_roll orientation!

```python
# Genesis lift_cube_env.py (lines 618, 687, 702, 791)
joint_targets[:, 4] = torch.pi / 2   # wrist_roll = +π/2  <-- WRONG

# Real robot requires (static finger on right side)
topdown_joints[4] = -np.pi / 2        # wrist_roll = -π/2  <-- CORRECT
```

**Impact**:
- Euler angles differ by ~π between sim and real
- Policy trained on wrong orientation cannot transfer
- `--fix_euler` transformation workaround did not help
- Checkpoint verified working in Genesis sim, fails on real robot

**Fix**: Retrain Genesis with correct wrist configuration:
```python
joint_targets[:, 4] = -torch.pi / 2  # Match real robot
```

## Status

- Retraining in progress with domain randomization and correct wrist config
- Current checkpoint (checkpoint_100352.pt) is NOT usable for real robot

## Files Modified

- `scripts/ppo_inference.py` - Main inference script with debug options
- `src/deploy/policy.py` - Added GenesisPPORunner class

## Next Steps

1. Print full low_dim_state and compare with Genesis values
2. Verify action interpretation in `cartesian_to_joints()`
3. Check if checkpoint actually achieves high success in Genesis eval
4. Consider running Genesis eval script to capture reference observations

## Sim2Real Diagnostic Test

Added `--sim_frames_dir` and `--sim_states_file` options to prove whether visual sim2real gap is the issue.

**Hypothesis**: If the policy produces correct actions with Genesis frames but wrong actions with real camera frames, the visual domain gap is the problem.

### Step 1: Record Genesis Episode

In Genesis eval, save frames and states from a successful episode:

```python
# In Genesis evaluation code, add saving logic:
import cv2
import os

save_dir = "genesis_successful_episode"
os.makedirs(save_dir, exist_ok=True)

for step in range(episode_length):
    obs = env.get_obs()  # dict with 'wrist_rgb' and 'low_dim_state'

    # Save frame as PNG (84x84 RGB)
    img_rgb = obs['wrist_rgb'].cpu().numpy().transpose(1, 2, 0)  # CHW -> HWC
    img_bgr = cv2.cvtColor((img_rgb * 255).astype(np.uint8), cv2.COLOR_RGB2BGR)
    cv2.imwrite(f"{save_dir}/frame_{step:04d}.png", img_bgr)

    # Save state
    states.append(obs['low_dim_state'].cpu().numpy())

# Save all states as single file
np.save(f"{save_dir}/states.npy", np.array(states))
```

### Step 2: Run Diagnostic on Real Robot

```bash
# Test with sim frames only (use real robot state)
uv run python scripts/ppo_inference.py \
    --checkpoint /path/to/checkpoint.pt \
    --sim_frames_dir ./genesis_successful_episode \
    --episode_length 50 \
    --debug_state

# Test with full sim observations (sim frames + sim states)
uv run python scripts/ppo_inference.py \
    --checkpoint /path/to/checkpoint.pt \
    --sim_frames_dir ./genesis_successful_episode \
    --sim_states_file ./genesis_successful_episode/states.npy \
    --dry_run \
    --episode_length 50 \
    --debug_state
```

### Expected Results

| Scenario | If Works | If Fails |
|----------|----------|----------|
| Sim frames + sim states (dry_run) | Policy architecture is correct | Bug in inference code |
| Sim frames + real states | Visual gap NOT the issue | State mismatch is the issue |
| Real frames + real states | Transfer works | Visual sim2real gap confirmed |

## Commands

```bash
# Basic run
uv run python scripts/ppo_inference.py \
    --checkpoint /path/to/checkpoint.pt \
    --camera_index 1 \
    --action_scale 0.02 \
    --episode_length 50

# With debug state output (recommended for debugging)
uv run python scripts/ppo_inference.py \
    --checkpoint /path/to/checkpoint.pt \
    --camera_index 1 \
    --debug_state \
    --episode_length 20

# Test Genesis wrist configuration
uv run python scripts/ppo_inference.py \
    --checkpoint /path/to/checkpoint.pt \
    --camera_index 1 \
    --genesis_wrist \
    --debug_state \
    --episode_length 20

# With visual debug output
uv run python scripts/ppo_inference.py \
    --checkpoint /path/to/checkpoint.pt \
    --camera_index 1 \
    --save_obs \
    --save_obs_video debug.mp4
```
