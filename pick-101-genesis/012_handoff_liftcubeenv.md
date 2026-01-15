# Handoff Document: LiftCubeEnv Training Environment

## Overview

This document describes the Genesis-based `LiftCubeEnv` used for training the SO-101 robot arm to lift a cube. The environment ports the MuJoCo `LiftCubeCartesianEnv` to Genesis simulator with Vulkan backend for AMD GPU support.

## Current State

**Training command:**
```bash
uv run python src/training/train_ppo.py --config-name multi_env_wrist
```

**Performance:** ~80 fps with 16 parallel envs, wrist camera rendering

**Known issues:**
- Eval is slow due to 650 physics steps per reset (3 episodes = ~2000 steps overhead)
- Success rate stuck at 0% - policy not learning to grasp yet

## Environment Architecture

### File: `src/envs/lift_cube_env.py`

**Action Space (4D continuous):**
- `[dx, dy, dz]` - Delta end-effector position (scaled by `action_scale=0.02`)
- `gripper` - Gripper command (-1 to 1)

**Observation Space (21D):**
- Joint positions (6): 5 arm + 1 gripper
- Joint velocities (6): 5 arm + 1 gripper
- Gripper position (3): XYZ world coords
- Gripper orientation (3): Euler angles (roll, pitch, yaw)
- Cube position (3): XYZ world coords

**Image Observation (for RL):**
- `rgb`: Wrist camera image `(C, H, W)` = `(3, 84, 84)` uint8
- `low_dim_state`: Proprioception `(18,)` - obs without cube_pos (privileged)

### DOF Layout (12 total)
| Index | DOF | Description |
|-------|-----|-------------|
| 0-5 | Cube free joint | tx, ty, tz, rx, ry, rz (uncontrolled) |
| 6-10 | Arm joints | shoulder_pan, shoulder_lift, elbow_flex, wrist_flex, wrist_roll |
| 11 | Gripper | Single DOF for both fingers |

### PD Control Gains
| DOFs | kp | kv |
|------|-----|-----|
| Cube (0-5) | 0 | 0 |
| Arm (6-10) | 1000 | 50 |
| Gripper (11) | 500 | 50 |

## Reset Behavior (curriculum_stage=3)

The reset involves 650 physics steps total to position the gripper at pre-grasp pose:

```
Phase 1: Arm initialization (100 steps)
  - Set wrist joints to pi/2 (top-down orientation)
  - Open gripper to 1.0
  - Zero cube velocity

Phase 2: Cube settle (50 steps)
  - Let physics settle

Phase 3: Move above cube (300 steps)
  - IK to position above cube (height_offset=0.03)
  - Gripper half-open (0.5)
  - locked_joints=[3,4] keeps wrist at pi/2

Phase 4: Move down to grasp height (200 steps)
  - IK to grasp position (grasp_z_offset=0.005)
  - Gripper stays half-open
```

**Pre-grasp offsets:**
- `height_offset = 0.03` - Height above cube for approach
- `grasp_z_offset = 0.005` - Final grasp height above cube center
- `finger_width_offset = -0.015` - X offset to center gripper on cube

## IK Controller

**File:** `src/controllers/jacobian_ik.py`

Uses Jacobian-based IK with:
- `gain=0.3` for smooth motion (during episode steps)
- `gain=0.5` for faster motion (during reset)
- `locked_joints=[3,4]` keeps wrist_flex and wrist_roll at pi/2 for top-down grasp
- Wrist joints explicitly set to pi/2 after IK computation

## Step Function

Each `step()`:
1. Clip action to [-1, 1]
2. Update target EE position: `target_ee_pos += delta_xyz * action_scale`
3. Clamp to workspace bounds
4. Compute IK with `gain=0.3`, `locked_joints=[3,4]`
5. Apply position control
6. Run 10 physics substeps
7. Compute reward using `reward_version` function
8. Check termination (success or timeout)

**Workspace bounds:**
- X: [0.05, 0.45] (sideways)
- Y: [-0.2, 0.35] (forward/back, robot faces -Y)
- Z: [0.01, 0.4] (up/down)

## Reward Function (v11)

**File:** `src/envs/rewards.py`

Components:
- Distance to cube reward
- Grasp reward (when gripper closes on cube)
- Lift reward (when cube height > lift_height)
- Hold reward (accumulated while lifted)
- Success bonus (when hold_steps reached)

## Training Configuration

**File:** `configs/multi_env_wrist.yaml`

```yaml
env:
  num_envs: 16
  use_wrist_cam: true
  max_episode_steps: 300
  action_scale: 0.02
  lift_height: 0.08
  hold_steps: 150
  reward_version: v11
  curriculum_stage: 3
  image_size: [84, 84]

train:
  total_timesteps: 1_000_000
  num_steps: 128  # per env (128 * 16 = 2048 total)
  save_freq: 50000
  eval_freq: 50000
  n_eval_episodes: 3
```

## Multi-Env Considerations

### Wrist Camera Rendering
- Single env: Camera attached to gripper link (efficient)
- Multi-env: Sequential rendering with manual camera positioning per env (AMD workaround)

### Eval/Video Recording
- Uses same training env (not separate instances)
- Only processes env 0's observations
- Broadcasts action to all envs: `action.expand(num_envs, -1)`
- **Reason:** Creating/destroying separate LiftCubeEnv corrupts EGL context

### Error Handling
- `evaluate()` wrapped in try-except, returns zeros on failure
- `record_video()` wrapped in try-except, logs warning on failure
- Entire eval block in training loop wrapped to prevent crashes

## Scene Configuration

**File:** `src/envs/scene_config.py`

Key positions:
- Robot: `(0.25, 0.314, 0.0)` - at table edge
- Cube: `~(0.25, 0.064, 0.015)` - on table in front of robot
- Table: Custom mesh at realistic height

Cameras:
- Wrist cam: Attached to gripper, FOV=90, 84x84
- Wide cam: For video recording, 480x480

## Recent Commits (Context)

```
42b16ad Fix EGL context corruption by reusing training env for eval/video
7c5337b Fix multi-env evaluation crash with proper error handling
1994f0e Use verified pre-grasp motion in training env reset
4f09d50 Fix arm motion and match MuJoCo pre-grasp sequence
82ad78d Fix multi-env support for wrist camera training
```

## Open Issues

1. **Success rate 0%** - Policy not learning to grasp
2. **Slow eval** - 650 steps per reset, 3 episodes = ~2000 steps overhead
3. **Contact detection** - Simplified, may not accurately detect grasp

## Next Steps

- Investigate why policy isn't learning (reward shaping, observation space)
- Consider reducing eval episodes or skipping elaborate reset for eval
- Improve contact detection for more accurate grasp reward
