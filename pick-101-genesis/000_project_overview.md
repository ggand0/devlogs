# Project Overview: pick-101-genesis

## Motivation

The original `pick-101` project achieved 100% success rate training image-based RL agents (DrQ-v2) for cube manipulation using MuJoCo simulation. However, real-world inference shows significant sim2real gap, primarily due to:

1. **Visual fidelity gap**: MuJoCo's OpenGL rasterization produces images that differ significantly from real camera feeds
2. **Physics mismatch**: Contact dynamics and friction behavior differ between simulation and reality
3. **Rendering limitations**: MuJoCo lacks photorealistic rendering capabilities needed for robust visual policies

## Why Genesis?

Genesis is a new physics platform that addresses these limitations:

| Feature | MuJoCo | Genesis |
|---------|--------|---------|
| Rendering | OpenGL rasterization | Native ray-tracing, photorealistic |
| GPU Simulation | CPU-bound physics | Full GPU acceleration |
| AMD GPU Support | N/A | Vulkan backend |
| Speed | ~10K FPS | ~43M FPS (claimed) |
| Physics Backends | Single (MuJoCo) | Multiple (rigid, MPM, SPH, FEM, PBD) |

Key advantages for sim2real:
- **Ray-traced rendering** should produce more realistic images
- **GPU-accelerated physics** enables faster domain randomization
- **Multi-backend physics** allows testing different contact models

## Project Structure

```
pick-101-genesis/
├── models/so101/           # MJCF robot models (copied from pick-101)
│   ├── lift_cube.xml       # Main scene definition
│   ├── so101_new_calib.xml # Robot model with finger pads
│   └── assets/             # STL mesh files
├── src/
│   ├── envs/
│   │   ├── lift_cube_env.py    # Genesis environment (to implement)
│   │   └── rewards/            # Reward functions (ported from pick-101)
│   ├── controllers/            # IK controller (to adapt)
│   └── training/               # Training scripts (to implement)
├── configs/                    # Training configurations
├── devlogs/                    # Development logs
└── tests/                      # Test scripts
```

## Hardware

- **GPU**: AMD Radeon RX 7900 XTX (24GB VRAM)
- **Backend**: Vulkan (for AMD GPU support)
- **PyTorch**: 2.9.1+rocm6.4

## Dependencies

- `genesis-world>=0.3.0` - Core simulation framework
- `torch>=2.8.0` (ROCm build) - Neural network training
- `gymnasium>=0.29.0` - Environment interface
- `omegaconf/hydra-core` - Configuration management

## Relationship to pick-101

This project is a **parallel implementation**, not a replacement:

- **pick-101**: MuJoCo-based, proven 100% success, baseline for comparison
- **pick-101-genesis**: Genesis-based, experimental, focused on sim2real

Portable components from pick-101:
- MJCF robot/scene models (Genesis supports MJCF)
- Reward functions (v11 for state-based, v19 for image-based)
- Training configurations (hyperparameters, curriculum stages)
- Task definition (lift cube to 8cm, hold for 3 seconds)

Components requiring rewrite:
- Environment wrapper (MuJoCo API → Genesis API)
- IK controller (MuJoCo Jacobian → Genesis equivalent)
- Training loop (RoboBase → Genesis OnPolicyRunner or custom)
- Camera/rendering pipeline (Genesis ray-tracing)

## Goals

1. **Phase 1**: Get basic simulation running with SO-101 model
2. **Phase 2**: Implement full environment with rewards and curriculum
3. **Phase 3**: Train state-based policy, validate against pick-101 baseline
4. **Phase 4**: Train image-based policy with ray-traced rendering
5. **Phase 5**: Real-world deployment, measure sim2real gap improvement

## References

- [Genesis GitHub](https://github.com/Genesis-Embodied-AI/Genesis)
- [Genesis Documentation](https://genesis-world.readthedocs.io/)
- [pick-101 repository](../pick-101/) - Original MuJoCo implementation
