# Devlog 064: Domain Randomization for Sim2Real

## Context

Deployed best DrQ-v2 model (800k checkpoint, 100% sim success) to real SO-101. Policy showed learned structure (moved down, closed gripper) but failed to complete the task. Root cause: visual domain gap between MuJoCo renders and real camera images.

**Sim vs Real visual differences:**
| Element | Sim | Real |
|---------|-----|------|
| Tabletop | Blue-gray checker | Beige |
| Robot base | Yellow | White |
| Static finger | Yellow | Orange |
| Lighting | Fixed | Variable |

## Implementation

Created `DomainRandomizationWrapper` that modifies MuJoCo scene at each reset:

### Scene-level (texture/material API)
- **Ground texture**: Directly modify `model.tex_data` to replace blue checker with solid beige/tan/gray (70%) or subtle checker (30%)
- **Skybox texture**: Randomize to gray/white/blue/warm gradients
- **Robot materials**: Yellow variations (65%) or white/gray (35%) via `model.mat_rgba`
- **Finger material**: Orange (35%) to match real robot
- **Lighting**: Vary `model.vis.headlight.diffuse` and `.ambient`

### Image-level (WristCameraWrapper)
- Brightness/contrast jitter
- Gaussian noise

## Files

- `src/envs/wrappers/domain_randomization.py` - Main wrapper
- `configs/image_based/drqv2_lift_dr.yaml` - Training config with DR enabled
- `scripts/test_domain_randomization.py` - Visual verification

## Usage

```bash
# Test randomization visually
python scripts/test_domain_randomization.py

# Train with DR
python -m src.training.train_image_rl --config-name drqv2_lift_dr
```

## MuJoCo Rendering Limitations

MuJoCo's renderer is primarily designed for **physics simulation**, not photorealistic rendering. Key limitations for image-based sim2real:

1. **Simple lighting model** - No global illumination, limited shadows
2. **Basic materials** - No PBR (physically-based rendering), limited texture support
3. **No real-world artifacts** - No lens distortion, chromatic aberration, motion blur
4. **Synthetic appearance** - Renders look distinctly "simulated"

### Alternatives for RGB-based Sim2Real

| Simulator | Rendering | Physics | Best For |
|-----------|-----------|---------|----------|
| **MuJoCo** | Basic OpenGL | Excellent | State-based RL, fast training |
| **ManiSkill** | Vulkan/ray-tracing | SAPIEN (good) | RGB sim2real, manipulation |
| **Isaac Sim** | RTX ray-tracing | PhysX 5 | Photorealistic, NVIDIA hardware |
| **SAPIEN** | Vulkan PBR | SAPIEN | Dexterous manipulation |

**ManiSkill** (https://github.com/haosulab/ManiSkill) is specifically designed for visual manipulation learning with:
- Photorealistic rendering via Vulkan
- Domain randomization built-in
- Real2Sim asset pipelines
- Better sim2real transfer for RGB observations

For this project, options going forward:
1. **Continue with MuJoCo + aggressive DR** - May work for simple tasks
2. **Port to ManiSkill** - Better visual fidelity, more work upfront
3. **HIL-SERL fine-tuning** - Use sim policy as initialization, fine-tune on real robot

## Next Steps

1. Train with DR config (~800k-1M steps)
2. Test zero-shot transfer on real robot
3. If still failing, consider ManiSkill port or HIL-SERL
