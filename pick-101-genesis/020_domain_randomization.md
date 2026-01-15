# Devlog 020: Domain Randomization for Sim2Real Transfer

**Date**: 2026-01-11
**Status**: Implemented

## Summary

Added comprehensive domain randomization to `LiftCubeEnv` to improve sim2real transfer. The implementation matches the techniques used in the [lerobot-sim2real](https://github.com/lerobot-sim2real) project which demonstrated zero-shot SO-101 grasp and lift.

## Motivation

The reality gap between simulation and real-world deployment is a major challenge for robot learning. Domain randomization addresses this by training on a distribution of environments, forcing the policy to learn features that are robust to visual and physical variations.

Key differences between sim and real:
- Lighting conditions (color, intensity, direction)
- Camera mounting position/orientation tolerances
- Object size/friction manufacturing tolerances
- Robot calibration errors

## Implementation Details

### Parameters Added to `LiftCubeEnv.__init__`

```python
# Domain randomization config (matching lerobot-sim2real)
domain_randomization: bool = True,
dr_cam_pos_noise: float = 0.025,      # ±2.5cm camera position noise
dr_cam_rot_noise: float = 0.005,      # ±0.005 rad camera rotation noise
dr_cam_fov_noise: float = 0.035,      # ±2° FOV noise (in radians) [NOT YET IMPLEMENTED]
dr_cube_size_range: tuple[float, float] = (0.024, 0.036),  # 24-36mm cube size
dr_friction_range: tuple[float, float] = (0.3, 1.5),      # Friction coefficient range
dr_ambient_range: tuple[float, float] = (0.08, 0.5),      # Ambient light per channel (lower for night)
dr_robot_pose_noise: float = 0.02,    # ±0.02 rad joint noise
dr_light_dir_noise: float = 0.5,      # ±0.5 noise on light direction x/y
dr_light_intensity_range: tuple[float, float] = (0.5, 1.4),  # Directional light intensity
dr_background_colors: tuple = (       # Day/night background options
    (0.914, 0.894, 0.890),  # Light (day)
    (0.08, 0.08, 0.10),     # Dark (night)
),
```

### Randomization Timing

| Parameter | When Randomized | Scope |
|-----------|----------------|-------|
| Ambient light | Scene creation | Per scene (fixed for episode) |
| Light direction | Scene creation | Per scene (fixed for episode) |
| Light intensity | Scene creation | Per scene (fixed for episode) |
| Background color | Scene creation | Per scene (fixed for episode) |
| Cube size | Scene creation | Per scene (fixed for episode) |
| Cube friction | Scene creation | Per scene (fixed for episode) |
| Camera pose noise | Every reset | Per environment |
| Robot initial pose | Every reset | Per environment |
| Cube XY position | Every reset | Per environment |

### 1. Ambient Light Randomization

Randomizes RGB channels independently to simulate different lighting conditions (warm/cool, bright/dim).

```python
if domain_randomization:
    ambient_r = np.random.uniform(dr_ambient_range[0], dr_ambient_range[1])
    ambient_g = np.random.uniform(dr_ambient_range[0], dr_ambient_range[1])
    ambient_b = np.random.uniform(dr_ambient_range[0], dr_ambient_range[1])
    ambient_light = (ambient_r, ambient_g, ambient_b)
```

**Range**: 0.08 - 0.5 per channel (lowered minimum for dim night conditions)
**Effect**: Creates warm (high R), cool (high B), or neutral lighting

### 2. Directional Light Direction

Randomizes the main light direction to simulate sun/overhead light from different angles.

```python
light_dir_x = np.random.uniform(-dr_light_dir_noise, dr_light_dir_noise)
light_dir_y = np.random.uniform(-dr_light_dir_noise, dr_light_dir_noise)
light_dir_z = -1.0  # Always pointing downward
# Normalize direction
light_dir_norm = np.sqrt(light_dir_x**2 + light_dir_y**2 + light_dir_z**2)
light_dir = (light_dir_x / light_dir_norm, light_dir_y / light_dir_norm, light_dir_z / light_dir_norm)
```

**Range**: ±0.5 in x/y (normalized with z=-1)
**Effect**: Shadows fall in different directions, highlights shift

### 3. Cube Size Randomization

Varies cube dimensions to handle real-world object size variations.

```python
cube_size = np.random.uniform(dr_cube_size_range[0], dr_cube_size_range[1])
```

**Range**: 24mm - 36mm (default real cube is ~30mm)
**Note**: Cube spawn Z is adjusted to `cube_size / 2` so it sits on the table

### 4. Friction Randomization

Varies cube-gripper friction to handle material/surface variations.

```python
cube_friction = np.random.uniform(dr_friction_range[0], dr_friction_range[1])
```

**Range**: 0.3 - 1.5
**Effect**: Low friction = cube slips more easily, high friction = easier to grasp

### 5. Camera Pose Noise

Per-environment noise on wrist camera position and orientation. Regenerated at each reset.

```python
def _randomize_camera_noise(self, env_ids: torch.Tensor = None):
    n = len(env_ids)
    self.cam_pos_noise[env_ids] = (torch.rand(n, 3, device=self.device) - 0.5) * 2 * self.dr_cam_pos_noise
    self.cam_rot_noise[env_ids] = (torch.rand(n, 3, device=self.device) - 0.5) * 2 * self.dr_cam_rot_noise

def _apply_camera_noise(self, cam_world_T: torch.Tensor, env_idx: int) -> torch.Tensor:
    noisy_T = cam_world_T.clone()
    noisy_T[:3, 3] += self.cam_pos_noise[env_idx]

    # Small angle approximation for rotation
    wx, wy, wz = self.cam_rot_noise[env_idx]
    R_noise = torch.tensor([
        [1, -wz, wy],
        [wz, 1, -wx],
        [-wy, wx, 1]
    ], device=self.device, dtype=torch.float32)
    noisy_T[:3, :3] = torch.matmul(R_noise, noisy_T[:3, :3])
    return noisy_T
```

**Position range**: ±2.5cm in each axis
**Rotation range**: ±0.005 rad (~0.3°) in each axis
**Effect**: Simulates camera mounting tolerances

### 6. Robot Initial Pose Noise

Adds noise to initial joint positions (first 3 joints only, wrist locked at π/2).

```python
if self.domain_randomization:
    pose_noise = (torch.rand(self.num_envs, 3, device=self.device) - 0.5) * 2 * self.dr_robot_pose_noise
    init_target[:, 0:3] += pose_noise  # shoulder_pan, shoulder_lift, elbow_flex
```

**Range**: ±0.02 rad (~1.1°) per joint
**Effect**: Robot starts from slightly different configurations

### 7. Cube XY Position Randomization

Randomizes cube spawn position at reset (already existed, documented here).

```python
xy_noise = (torch.rand(n_reset, 2, device=self.device) - 0.5) * 0.10  # ±5cm
cube_spawn_batch[:, :2] += xy_noise
```

**Range**: ±5cm in X and Y
**Effect**: Cube appears in different locations relative to robot

### 8. Background Color Randomization

Simulates day vs night room conditions by randomly selecting background color.

```python
dr_background_colors: tuple = (
    (0.914, 0.894, 0.890),  # Light (day) - off-white
    (0.08, 0.08, 0.10),     # Dark (night) - near black
)
bg_idx = np.random.randint(len(dr_background_colors))
background_color = dr_background_colors[bg_idx]
```

**Options**: Light (off-white) or Dark (near-black)
**Effect**: Simulates daytime vs nighttime room lighting

### 9. Light Intensity Randomization

Simulates different ceiling light configurations (e.g., one vs two lights on).

```python
dr_light_intensity_range: tuple[float, float] = (0.5, 1.4)
light_intensity = np.random.uniform(dr_light_intensity_range[0], dr_light_intensity_range[1])
```

**Range**: 0.5 - 1.4
**Effect**: Low intensity = single/dim ceiling light, high intensity = bright overhead lighting

Combined with the lowered ambient range (0.08 - 0.5), this allows for very dark night scenes.

## Comparison with lerobot-sim2real

| Feature | lerobot-sim2real | pick-101-genesis |
|---------|-----------------|------------------|
| Simulator | ManiSkill 3 (SAPIEN) | Genesis |
| Camera noise | ±2.5cm pos, ±0.005 rad rot, ±2° FOV | ±2.5cm pos, ±0.005 rad rot (FOV TODO) |
| Cube size | 24-36mm | 24-36mm |
| Friction | 0.3-1.5 | 0.3-1.5 |
| Ambient light | 0.2-0.5 per channel | 0.08-0.5 per channel (darker for night) |
| Light intensity | Not documented | 0.5-1.4 (simulates ceiling lights) |
| Robot pose | ±0.02 rad | ±0.02 rad (first 3 joints) |
| Light direction | Not documented | ±0.5 x/y noise |
| Background color | Greenscreen | Light/dark randomization (day/night) |

## Demo Script

`tests/demo_domain_randomization.py` generates videos showing DR variations:

```bash
uv run python tests/demo_domain_randomization.py
```

Output:
- `tests/dr_demos/dr_demo_1.mp4` through `dr_demo_5.mp4` - Full grasp attempts
- `tests/dr_demos/dr_comparison_grid.png` - 2x2 grid showing variations

## Future Work

1. **FOV randomization**: Parameter exists but not yet implemented (requires camera recreation)
2. **Texture randomization**: Vary table/cube textures
3. **Dynamics randomization**: Vary robot dynamics (damping, friction)

## References

- [lerobot-sim2real](https://github.com/lerobot-sim2real) - Zero-shot SO-101 grasp with ManiSkill
- [Domain Randomization for Sim2Real Transfer](https://arxiv.org/abs/1703.06907) - Original DR paper
- [OpenAI Rubik's Cube](https://openai.com/research/solving-rubiks-cube) - Extensive DR for dexterous manipulation
