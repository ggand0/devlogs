# Devlog 025: Per-Reset Domain Randomization Fix

**Date:** 2026-01-13

## Critical Bug: DR Parameters Shared Across All Envs

### Problem

Domain randomization for cube size, friction, and lighting was completely broken in multi-env training. All 16 parallel environments shared **identical** DR parameters because they were sampled only once at scene creation.

```python
# OLD (BROKEN) CODE - in __init__
if domain_randomization:
    cube_size = np.random.uniform(dr_cube_size_range[0], dr_cube_size_range[1])
    cube_friction = np.random.uniform(dr_friction_range[0], dr_friction_range[1])
    ambient_light = (np.random.uniform(...), ...)
    light_intensity = np.random.uniform(...)
```

This meant:
- All 16 envs had the **exact same** cube size
- All 16 envs had the **exact same** friction
- All 16 envs had the **exact same** lighting
- These values **never changed** throughout training

### Impact

Zero effective domain randomization during training. The agent only ever saw ONE configuration, defeating the entire purpose of DR for sim2real transfer.

## Root Cause: Genesis Scene Architecture

Genesis creates a single scene with `num_envs` parallel instances. Unlike MuJoCo where you can modify object properties at runtime:

1. **Mesh sizes are fixed after `scene.build()`** - Cannot call `cube.set_size(new_size)`
2. **Lighting is scene-level** - One light configuration shared by all envs
3. **Material properties (friction) are fixed** - Cannot modify after entity creation

## Solution

### 1. Multi-Cube Entity Swapping (Cube Size/Friction DR)

Since mesh sizes can't change after build, create multiple cube variants at initialization and swap which one is "active" per environment on reset.

**At init - create 4 cube variants:**
```python
# Create 4 cube size variants spanning the DR range
self.cube_sizes = np.linspace(dr_cube_size_range[0], dr_cube_size_range[1], 4)
# Sizes: [24mm, 28mm, 32mm, 36mm]

# Sample friction for each variant
self.cube_frictions = np.random.uniform(dr_friction_range[0], dr_friction_range[1], 4)

# Create all cube entities
self.cubes = []
for i, (cube_size, cube_friction) in enumerate(zip(self.cube_sizes, self.cube_frictions)):
    init_z = cube_size / 2 if i == 0 else -1.0  # First at spawn, others hidden
    cube = self.scene.add_entity(
        gs.morphs.Box(size=(cube_size, cube_size, cube_size), pos=(0.25, 0.0, init_z)),
        material=gs.materials.Rigid(friction=cube_friction),
    )
    self.cubes.append(cube)

# Track active cube per env
self.active_cube_idx = torch.zeros(num_envs, dtype=torch.long, device=device)
```

**On reset - swap cubes:**
```python
# Randomly assign cube variant to each resetting env
new_cube_idx = torch.randint(0, num_cube_variants, (n_reset,), device=self.device)
self.active_cube_idx[env_ids] = new_cube_idx

# Position active cubes at spawn, hide inactive ones at z=-1
for cube_idx, cube in enumerate(self.cubes):
    for env_id in env_ids:
        if self.active_cube_idx[env_id] == cube_idx:
            # Active: position at spawn with XY noise
            current_pos[env_id, :2] = base_xy + xy_noise
            current_pos[env_id, 2] = cube_size / 2
        else:
            # Inactive: hide below table
            current_pos[env_id, 2] = -1.0
```

**Helper method for correct cube position:**
```python
def _get_active_cube_pos(self) -> torch.Tensor:
    """Get position of the active cube for each environment."""
    all_cube_pos = torch.stack([cube.get_pos() for cube in self.cubes], dim=1)
    idx = self.active_cube_idx.unsqueeze(-1).unsqueeze(-1).expand(-1, -1, 3)
    return torch.gather(all_cube_pos, 1, idx).squeeze(1)
```

### 2. Runtime Lighting Modification (Lighting DR)

Genesis uses pyrender for rasterization rendering. Pyrender scene properties CAN be modified at runtime.

**Access path to pyrender scene:**
```
scene._visualizer._context._scene  ->  pyrender.Scene
```

**Lighting randomization on reset:**
```python
def _randomize_lighting(self):
    # Access pyrender scene through Genesis internals
    visualizer = self.scene._visualizer
    context = visualizer._context
    pyrender_scene = context._scene

    # Randomize ambient light (RGB independently)
    pyrender_scene.ambient_light = np.array([
        np.random.uniform(0.08, 0.5),  # R
        np.random.uniform(0.08, 0.5),  # G
        np.random.uniform(0.08, 0.5),  # B
    ], dtype=np.float32)

    # Randomize directional lights
    for light_node in pyrender_scene.directional_light_nodes:
        # Randomize intensity
        light_node.light.intensity = np.random.uniform(0.5, 1.4)

        # Randomize direction (always pointing down with XY noise)
        light_dir = np.array([
            np.random.uniform(-0.5, 0.5),  # X noise
            np.random.uniform(-0.5, 0.5),  # Y noise
            -1.0  # Always down
        ])
        light_dir = light_dir / np.linalg.norm(light_dir)

        # Convert direction to pose matrix
        from genesis.utils import geom as gu
        pose = np.eye(4, dtype=np.float32)
        gu.z_up_to_R(-light_dir, out=pose[:3, :3])
        light_node.matrix = pose
```

## DR Parameters

| Parameter | Range | Variants |
|-----------|-------|----------|
| Cube size | 24-36mm | 4 discrete (24, 28, 32, 36mm) |
| Friction | 0.3-1.5 | 4 values (one per cube) |
| Ambient light | 0.08-0.5 per channel | Continuous |
| Light intensity | 0.5-1.4 | Continuous |
| Light direction | ±0.5 XY noise | Continuous |
| Cube XY position | ±5cm | Continuous |
| Camera position | ±2.5cm | Continuous per-env |
| Camera rotation | ±0.005 rad | Continuous per-env |
| Robot initial pose | ±0.02 rad | Continuous |

## Testing

### Verification Script

```python
import genesis as gs
gs.init(backend=gs.vulkan, logging_level='warning')
from src.envs.lift_cube_env import LiftCubeEnv

env = LiftCubeEnv(num_envs=1, use_wrist_cam=True, domain_randomization=True)

# Access pyrender scene
pyrender_scene = env.scene._visualizer._context._scene

print('Initial ambient:', pyrender_scene.ambient_light)
print('Initial intensity:', list(pyrender_scene.directional_light_nodes)[0].light.intensity)

env.reset()
print('After reset ambient:', pyrender_scene.ambient_light)  # Different!
print('After reset intensity:', ...)  # Different!
```

### Test Results

```
Initial ambient: [0.25, 0.27, 0.39], intensity: 0.55
After reset:     [0.11, 0.41, 0.38], intensity: 0.93  <- Changed!
After 2nd reset: [0.26, 0.32, 0.40], intensity: 0.96  <- Changed again!
```

### Visual Verification

Save sample images to verify lighting differences:
```bash
uv run python -c "
import imageio
import genesis as gs
gs.init(backend=gs.vulkan, logging_level='warning')
from src.envs.lift_cube_env import LiftCubeEnv

env = LiftCubeEnv(num_envs=1, use_wrist_cam=True, domain_randomization=True)
for i in range(5):
    env.reset()
    img = env.get_wrist_camera_obs()
    imageio.imwrite(f'lighting_test_{i}.png', img)
env.close()
"
```

Compare the 5 images - should show noticeably different lighting conditions.

## Limitations

### Lighting is Scene-Level

All 16 parallel envs see the **same lighting** at any moment. Lighting only changes on reset. This is a Genesis limitation - pyrender scene is shared.

**Mitigation:** Lighting changes on every reset (any env), so across training the agent sees thousands of different lighting conditions.

### Cube Variants are Discrete

Only 4 cube sizes (24, 28, 32, 36mm) instead of continuous range. This is because Genesis can't modify mesh sizes after `scene.build()`.

**Mitigation:** 4 variants still provide meaningful size variation for sim2real. More variants would increase memory usage.

## Files Modified

- `src/envs/lift_cube_env.py`:
  - Multi-cube creation at init
  - `active_cube_idx` and `per_env_cube_size` tracking tensors
  - `_get_active_cube_pos()` helper method
  - `_randomize_lighting()` method
  - Updated `reset()` to swap cubes and randomize lighting
  - Updated `_check_cube_contacts()` for multi-cube
  - Updated all `self.cube.get_pos()` to `_get_active_cube_pos()`

## Commit

`4974862` - Add per-reset domain randomization for cube size and lighting

## Verification During Training

1. **Eval videos** should show visible lighting differences between episodes
2. **TensorBoard** should show varied episode performance (due to different cube sizes)
3. **Cube size distribution** logged via `per_env_cube_size` tensor

## Next Steps

1. Run training with fixed DR to verify sim2real transfer improvement
2. Consider adding more cube variants if memory allows
3. Monitor eval success across different lighting conditions
