# Devlog 065: ManiSkill Port for SO-101

## Context

After implementing domain randomization in MuJoCo (devlog 064), it became clear that MuJoCo's OpenGL-based rendering is not ideal for RGB-based sim2real. ManiSkill uses SAPIEN with Vulkan rendering, providing much better visual fidelity for zero-shot sim2real transfer.

The lerobot-sim2real repo already has a working pipeline for SO-100 with ManiSkill + PPO. We ported SO-101 to this setup.

## Implementation

### Location

All files added to `/home/gota/ggando/ml/lerobot-sim2real`:

```
lerobot-sim2real/
├── assets/so101/
│   ├── so101.urdf              # Converted from pick-101 MJCF
│   └── meshes -> pick-101/...  # Symlink to existing meshes
├── lerobot_sim2real/
│   ├── agents/
│   │   ├── __init__.py
│   │   └── so101.py            # Robot agent definition
│   ├── envs/
│   │   ├── __init__.py
│   │   └── so101_lift_cube.py  # ManiSkill environment
│   └── scripts/
│       └── test_so101_env.py   # Verification script
├── env_config_so101.json       # Training config
└── pyproject.toml              # uv project config
```

### Key Components

**1. SO101 Agent (`agents/so101.py`)**
- Inherits from ManiSkill's `BaseAgent`
- Controller configs: `pd_joint_delta_pos` (default), `pd_joint_pos`, `pd_joint_target_delta_pos`
- Grasping detection using contact forces between `gripper_link` and `moving_jaw_so101_v1_link`
- TCP computed as midpoint between finger links

**2. SO101LiftCube Environment (`envs/so101_lift_cube.py`)**
- Inherits from `BaseDigitalTwinEnv` for greenscreen support
- Uses `TableSceneBuilder` for scene (table + floor)
- Domain randomization:
  - Camera position/FOV noise
  - Cube size, friction, color
  - Robot color
  - Lighting intensity
- Reward function ported from pick-101 v19

**3. Training Config (`env_config_so101.json`)**
```json
{
  "base_camera_settings": {
    "pos": [0.5, 0.3, 0.3],
    "fov": 0.86,
    "target": [0.25, 0.0, 0.0]
  },
  "spawn_box_pos": [0.25, 0.0],
  "spawn_box_half_size": 0.03,
  "domain_randomization_config": {
    "cube_color": "red",
    "ground_color": "random"
  }
}
```

## Test Results

```
Action space: Box(-1.0, 1.0, (6,), float32)
Observation space: Dict with RGB 128x128 images
Reset: successful
Step: successful
Render: 512x512 frames
```

## Training Command

```bash
cd /home/gota/ggando/ml/lerobot-sim2real

uv run python lerobot_sim2real/scripts/train_ppo_rgb.py \
  --env-id="SO101LiftCube-v1" \
  --env-kwargs-json-path=env_config_so101.json \
  --ppo.num_envs=1024 \
  --ppo.num-steps=16 \
  --ppo.update_epochs=8 \
  --ppo.num_minibatches=32 \
  --ppo.total_timesteps=50_000_000 \
  --ppo.gamma=0.9 \
  --ppo.num_eval_envs=16 \
  --ppo.num-eval-steps=64 \
  --ppo.exp-name="ppo-SO101LiftCube-v1-rgb" \
  --ppo.track --ppo.wandb_project_name "SO101-ManiSkill"
```

## Why ManiSkill over MuJoCo for RGB Sim2Real

| Aspect | MuJoCo | ManiSkill |
|--------|--------|-----------|
| Renderer | OpenGL (legacy) | Vulkan (modern) |
| PBR Materials | No | Yes |
| Ray tracing | No | Optional |
| GPU parallelization | Limited | Native (1000s of envs) |
| Greenscreen | Manual texture hacking | Built-in overlay system |
| Sim2real track record | State-based | RGB zero-shot proven |

## Next Steps

1. Train with PPO using lerobot-sim2real infrastructure
2. Compare learning curves with MuJoCo DrQ-v2
3. Test real robot deployment with trained checkpoint
4. If needed, integrate DrQ-v2 into ManiSkill later
