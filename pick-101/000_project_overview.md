# SO-101 Lift Task RL Project Overview

## Goal

Train the SO-101 robot arm to pick up a cube and lift it to a target height (8cm) using reinforcement learning in MuJoCo simulation.

## Stack

| Component | Package | Version |
|-----------|---------|---------|
| Physics | MuJoCo | 3.4.0 |
| RL Framework | Stable-Baselines3 | 2.7.1 |
| Gym Interface | Gymnasium | 1.2.2 |
| Deep Learning | PyTorch + ROCm 6.4 | 2.9.1 |
| GPU | AMD Radeon RX 7900 XTX | - |

## Image-Based RL Resources

### Papers
- **DrQ-v2** - [Mastering Visual Continuous Control: Improved Data-Augmented RL](https://arxiv.org/abs/2107.09645) - State-of-the-art pixel-based RL with data augmentation
- **DrQ** - [Image Augmentation Is All You Need](https://arxiv.org/abs/2004.13649) - Random shift augmentation for pixel-based RL
- **SAC-AE** - [Improving SAC with Autoencoders](https://arxiv.org/abs/1910.01741) - SAC from pixels with reconstruction loss
- **CURL** - [Contrastive Unsupervised Representations for RL](https://arxiv.org/abs/2004.04136) - Contrastive learning for visual RL
- **RAD** - [Reinforcement Learning with Augmented Data](https://arxiv.org/abs/2004.14990) - Systematic study of data augmentation

### Frameworks
- **RoboBase** - https://github.com/SiddhantTandon/roboBase - DrQ-v2 implementation used in this project
- **DMControl** - https://github.com/deepmind/dm_control - DeepMind's physics-based control suite

### Key Techniques
- Frame stacking (3 frames) for temporal information
- Random shift augmentation (±4 pixels) for generalization
- Separate encoder for actor/critic with EMA target
- Low-dim state (proprioception) concatenated with visual features

## Project Structure

```
explore-mujoco/
├── models/so101/                 # Robot models and scenes
│   ├── so101_new_calib.xml       # Current robot (finger pads, fingertip sites)
│   ├── so101_ik_grasp.xml        # Legacy robot (no pads)
│   ├── so101_horizontal_grasp.xml # Horizontal approach variant
│   ├── lift_cube.xml             # Scene: elliptic friction
│   ├── lift_cube_ik_grasp.xml    # Scene: default friction
│   ├── lift_cube_horizontal.xml  # Scene: horizontal approach
│   └── assets/                   # STL mesh files (Git LFS)
├── src/controllers/
│   └── ik_controller.py          # Damped least-squares IK
├── envs/
│   ├── lift_cube.py              # Main env: Cartesian action space
│   └── pick_cube.py              # Original joint-space env
├── tests/                        # Test scripts
├── runs/                         # Training outputs
└── devlogs/                      # Experiment documentation
```

## Environment Design

### Observation Space (21 dims)
- Joint positions (6)
- Joint velocities (6)
- Gripper position (3)
- Gripper orientation as euler angles (3)
- Cube position (3)

### Action Space (4 dims) - Cartesian
- Delta X, Y, Z for end-effector (scaled by 2cm/step)
- Gripper open/close (-1 to 1)

IK controller internally converts Cartesian targets to joint controls.

### Success Criteria
- Cube lifted to z > 0.08m (8cm)
- While being grasped (pinched between both gripper parts)
- Held for 10 consecutive steps

## Experiment History

| Version | Reward Structure | Result | Key Finding |
|---------|-----------------|--------|-------------|
| v1 | Reach + binary grasp + binary lift | 0% success | Grasp works, but pushes cube down |
| v2 | Reach + continuous lift (no grasp condition) | 0% success | Disrupted grasping entirely |
| v3 | Reach + grasp bonus + conditional lift | 0% success | Same local optimum as v1 |
| v4 | Reach + grasp-only-when-lifted | 0% success | Agent stopped grasping entirely |

### The Core Problem

Every reward structure has been exploited:
1. **Top-down approach**: Agent approaches from above, contacts cube, but can only push down
2. **Local optimum**: Reach reward (~0.9/step) + grasp bonus (~0.25/step) gives stable ~225 reward per episode
3. **No lift gradient**: Cube at z=0.007 (pushed down) gets same reward as z=0.010 (baseline)
4. **Grasp not required**: When grasp bonus removed, agent just hovers with open gripper

## Key Checkpoints

```
# V1 Original (500k) - learns to grasp but not lift
runs/lift_cube_cartesian/20251214_180426/checkpoints/sac_lift_cartesian_500000_steps.zip

# V1 Extended (1M) - same behavior, more confident
runs/lift_cube_cartesian/20251215_090951_resumed/checkpoints/sac_lift_cartesian_1000000_steps.zip

# V3 Fresh (500k) - still converges to push-down
runs/lift_cube/20251215_160643/checkpoints/sac_lift_500000_steps.zip

# V4 (500k) - hovers with open gripper, never grasps
runs/lift_cube/20251215_190304/checkpoints/sac_lift_500000_steps.zip
```

## Current Reward Structure (V4 - in lift_cube.py)

```python
def _compute_reward(self, info):
    reward = 0.0
    cube_z = info["cube_z"]

    # Reach: encourage gripper to approach cube
    reach_reward = 1.0 - np.tanh(10.0 * info["gripper_to_cube"])
    reward += reach_reward

    # Small lift reward for exploration (even without grasp)
    lift_baseline = max(0, (cube_z - 0.01) * 10.0)
    reward += lift_baseline

    # Grasp bonus ONLY when cube is elevated
    if info["is_grasping"] and cube_z > 0.01:
        reward += 0.25
        reward += (cube_z - 0.01) * 40.0

    # Binary lift bonus
    if cube_z > 0.02:
        reward += 1.0

    # Success bonus
    if info["is_success"]:
        reward += 10.0

    return reward
```

## Commands

```bash
# Train for 500k steps
uv run python train_lift.py --timesteps 500000

# Resume training from checkpoint
uv run python train_lift.py --resume runs/lift_cube/TIMESTAMP --timesteps 500000

# Evaluate with video
uv run python eval_cartesian.py \
    --model runs/lift_cube/TIMESTAMP/checkpoints/sac_lift_500000_steps.zip \
    --normalize runs/lift_cube/TIMESTAMP/vec_normalize.pkl \
    --episodes 5
```

## Next Steps to Try

1. **V5: Negative reward for pushing down** - Penalize cube_z < 0.01
2. **Curriculum: elevated cube** - Spawn cube at z=0.03-0.05 sometimes
3. **HER (Hindsight Experience Replay)** - Goal-conditioned learning
4. **Approach angle constraint** - Force side approach in early training
5. **Increase cube friction** - Make grasping more forgiving

## Devlogs

- [001_initial-setup.md](001_initial-setup.md) - Project setup, joint-space env
- [002_kernel-upgrade-training.md](002_kernel-upgrade-training.md) - ROCm/kernel issues
- [003_curriculum-research.md](003_curriculum-research.md) - Curriculum learning research
- [004_cartesian-ik-training.md](004_cartesian-ik-training.md) - IK controller, Cartesian action space
- [005_continuous-lift-reward.md](005_continuous-lift-reward.md) - V2 experiment, resume feature
- [006_v1-to-1M-and-staged-rewards.md](006_v1-to-1M-and-staged-rewards.md) - V1 to 1M, V3, V4 experiments
- [011_graspframe_fix.md](011_graspframe_fix.md) - Graspframe site positioning
- [013_collision_exclusions.md](013_collision_exclusions.md) - Gripper self-collision fix
- [014_ik_orientation_control.md](014_ik_orientation_control.md) - IK orientation + locked joints
- [015_gripperframe_centering.md](015_gripperframe_centering.md) - Gripperframe Y-axis centering
- [018_horizontal_grasp.md](018_horizontal_grasp.md) - Horizontal approach exploration
- [019_elliptic_cone_friction.md](019_elliptic_cone_friction.md) - Friction tuning for slip prevention
- [020_topdown_pick.md](020_topdown_pick.md) - Top-down pick with fingertip targeting
- [021_finger_pad_collision_fix.md](021_finger_pad_collision_fix.md) - Finger pad boxes for stable grasping
- [022_test_setups.md](022_test_setups.md) - Test script configurations (3 setups)
- [023_model_reorganization.md](023_model_reorganization.md) - Move models out of submodule, Git LFS

## Test Configurations

Three distinct test setups exist in `models/so101/`:

| Setup | Robot Model | Scene | Script | Features |
|-------|-------------|-------|--------|----------|
| Top-Down (current) | so101_new_calib.xml | lift_cube.xml | test_topdown_pick.py | Finger pads, elliptic friction |
| IK Grasp (legacy) | so101_ik_grasp.xml | lift_cube_ik_grasp.xml | test_ik_grasp*.py | Original model, default friction |
| Horizontal (experimental) | so101_horizontal_grasp.xml | lift_cube_horizontal.xml | test_horizontal_grasp.py | Horizontal approach |

See [022_test_setups.md](022_test_setups.md) for detailed documentation.

### Quick Commands

```bash
# Top-down pick (current best)
PYTHONPATH=. uv run python tests/test_topdown_pick.py

# IK grasp (legacy, reproduces original behavior)
PYTHONPATH=. uv run python tests/test_ik_grasp.py
PYTHONPATH=. uv run python tests/test_ik_grasp_video.py

# Horizontal grasp (experimental)
PYTHONPATH=. uv run python tests/test_horizontal_grasp.py
```
