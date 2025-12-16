# Initial Setup: SO-101 Pick-and-Place RL

## Overview

Set up a reinforcement learning environment for training the SO-101 robot arm to perform a pick-and-place task in MuJoCo simulation.

## Stack

| Component | Package | Version |
|-----------|---------|---------|
| Physics | MuJoCo | 3.4.0 |
| RL Framework | Stable-Baselines3 | 2.7.1 |
| Gym Interface | Gymnasium | 1.2.2 |
| Deep Learning | PyTorch + ROCm 6.4 | 2.9.1 |
| GPU | AMD Radeon RX 7900 XTX | - |

## Project Structure

```
explore-mujoco/
├── envs/
│   ├── __init__.py
│   └── pick_cube.py          # Custom Gymnasium environment
├── examples/
│   ├── 01_view_so101.py      # View arm in MuJoCo viewer
│   ├── 02_random_actions.py  # Random control demo
│   ├── 03_render_video.py    # Headless video render
│   └── 04_view_pick_cube.py  # View pick-cube scene
├── docs/
│   └── mujoco-viewer-controls.md
├── train.py                  # SAC training script
├── eval.py                   # Evaluation with viewer
└── SO-ARM100/                # Cloned robot model repo (not tracked)
    └── Simulation/SO101/
        └── pick_cube_scene.xml  # Custom scene with cube + target
```

## Environment Design

### Observation Space (21 dims)
- Joint positions (6)
- Joint velocities (6)
- Gripper position (3)
- Cube position (3)
- Target position (3)

### Action Space (6 dims)
- Delta joint positions, scaled by 0.1
- Clipped to actuator control ranges

### Reward Function
```
reward = -gripper_to_cube - 2.0 * cube_to_target + success_bonus + drop_penalty
```
- Encourages reaching the cube first, then moving it to target
- +10 bonus when cube within 3cm of target
- -5 penalty if cube falls off table

### Task Setup
- Cube spawns at (0.40, -0.10, 0.015) with slight randomization
- Target at (0.40, 0.10, 0.015) - 20cm away in Y direction
- Both positions within arm's workspace (~0.35-0.47m in X)

## Training Results (100k steps, ~20 min)

```
Success rate: ~1%
Mean reward: -77.5
Episodes: ~500
```

The arm learns to reach toward the cube but doesn't reliably grasp and move it. This is expected - pick-and-place typically needs:
- 500k-1M+ timesteps
- Staged rewards (reach → grasp → lift → place)
- Possibly curriculum learning

## Key Learnings

1. **Mesh paths matter** - MuJoCo XML files need meshes in correct relative paths. Scene XML placed in SO-ARM100 repo to share assets.

2. **Arm workspace** - Initial cube/target positions were unreachable. Had to analyze gripper positions at various joint configs to find valid workspace.

3. **VecNormalize sync** - For eval rendering, must wrap the same env instance in DummyVecEnv so viewer sees physics updates.

4. **Real-time pacing** - Added 20ms sleep per step in eval to make motion visible in viewer.

## Next Steps

- [ ] Train longer (500k+ steps)
- [ ] Implement staged rewards
- [ ] Add grasp detection (contact sensors)
- [ ] Try TD3 or PPO for comparison
- [ ] Curriculum: start with cube closer to target
