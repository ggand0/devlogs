# Wrist Camera and PPO Training Setup

## Summary

Added wrist camera support to LiftCubeEnv and implemented PPO training for both image-based and state-based RL.

## Wrist Camera Implementation

### Configuration
Based on MuJoCo wrist_cam from `so101_new_calib.xml`:
- Position: `x=0.02, y=-0.08, z=-0.02` (relative to gripper)
- Original quaternion: `(0, 0, -0.2164, 0.9763)` → yaw=180°, pitch=-25°
- FOV: 86° vertical

### Genesis Camera Attachment
Genesis cameras use `camera.attach(link, offset_T)` where offset_T is a 4x4 transformation matrix.

```python
# Genesis camera looks along -Z by default (OpenGL convention)
# Rotation to look down at workspace from gripper:
Ry = 90°   # Look along Y axis (toward workspace)
Rx = -25°  # Tilt down slightly

offset_T = eye(4)
offset_T[:3, :3] = Ry @ Rx
offset_T[:3, 3] = [0.02, -0.08, -0.02]
camera.attach(gripper_link, offset_T)
```

### Current View
Camera shows workspace from above gripper - cube visible but angle may need adjustment. The MuJoCo camera faced "forward" from gripper fingers; Genesis orientation differs.

**TODO**: Tweak camera rotation to better match MuJoCo wrist_cam perspective.

## PPO Training

### Two Variants

1. **Image-based** (`ppo.py`, `train_ppo.py`)
   - CNN encoder (Nature architecture) for 84x84 RGB
   - Combined with low-dim proprioception (18D)
   - Slow: ~9.5 steps/sec due to rendering overhead

2. **State-based** (`ppo_state.py`, `train_ppo_state.py`)
   - MLP encoder for 21D state observations
   - No camera rendering
   - Faster: ~65 steps/sec (7x faster)

### Performance Issue
Image-based training is slow because camera.render() is called every step:
- With rendering: ~9.5 steps/sec (~3 hours for 100k steps)
- Without rendering: ~65 steps/sec (~25 minutes for 100k steps)

Genesis physics FPS is ~500+ without rendering bottleneck. The wrist camera rendering adds significant overhead.

### Usage

```bash
# Fast state-based training (no images)
uv run python src/training/train_ppo_state.py --total_timesteps 100000

# Image-based training (slow but uses wrist cam)
uv run python src/training/train_ppo.py --total_timesteps 100000
```

## Network Architectures

### Image-based ActorCritic
```
CNN Encoder (84x84x3):
  Conv2d(3, 32, 8, stride=4) -> ReLU
  Conv2d(32, 64, 4, stride=2) -> ReLU  
  Conv2d(64, 64, 3, stride=1) -> ReLU
  Flatten -> Linear(3136, 256)

Low-dim Encoder (18D):
  Linear(18, 64) -> ReLU -> Linear(64, 64) -> ReLU

Combined (256 + 64 = 320):
  Actor: Linear(320, 256) -> ReLU -> Linear(256, 4)
  Critic: Linear(320, 256) -> ReLU -> Linear(256, 1)
```

### State-based ActorCritic
```
Encoder (21D):
  Linear(21, 256) -> Tanh -> Linear(256, 256) -> Tanh

Actor: Linear(256, 4)
Critic: Linear(256, 1)
```

## Files Added
- `src/envs/lift_cube_env.py`: Added wrist camera support
- `src/training/ppo.py`: Image-based PPO agent
- `src/training/ppo_state.py`: State-based PPO agent
- `src/training/train_ppo.py`: Image-based training script
- `src/training/train_ppo_state.py`: State-based training script
- `tests/test_wrist_camera.py`: Camera test

## Next Steps

1. **Optimize image-based training**:
   - Skip rendering some frames (frame stacking)
   - Use more parallel environments
   - Consider async rendering

2. **Tune camera angle** to better match MuJoCo view

3. **Verify reward signal** with state-based training first

4. **Scale up training** once basic setup works
