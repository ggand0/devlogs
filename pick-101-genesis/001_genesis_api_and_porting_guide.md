# Genesis API and Porting Guide

## Genesis Initialization

```python
import genesis as gs

# Initialize with Vulkan backend (required for AMD GPUs)
gs.init(backend=gs.vulkan)

# Create scene with physics and viewer options
scene = gs.Scene(
    sim_options=gs.options.SimOptions(dt=0.002),  # 500Hz physics
    viewer_options=gs.options.ViewerOptions(
        camera_pos=(1.0, 1.0, 1.0),
        camera_lookat=(0.0, 0.0, 0.3),
    ),
    show_viewer=True,  # Set False for headless training
)
```

## Loading MJCF Models

Genesis supports MJCF format directly:

```python
# Load robot from MJCF file
robot = scene.add_entity(
    gs.morphs.MJCF(file="models/so101/lift_cube.xml"),
)

# Build the scene (must be called before simulation)
scene.build()
```

## Key API Differences: MuJoCo vs Genesis

### Joint Access

**MuJoCo:**
```python
joint_pos = data.qpos[joint_ids]
joint_vel = data.qvel[joint_ids]
mujoco.mj_step(model, data)
```

**Genesis:**
```python
joint_pos = robot.get_dofs_position()
joint_vel = robot.get_dofs_velocity()
scene.step()
```

### Body/Site Positions

**MuJoCo:**
```python
site_id = mujoco.mj_name2id(model, mujoco.mjtObj.mjOBJ_SITE, "gripperframe")
pos = data.site_xpos[site_id]
```

**Genesis:**
```python
# Need to find Genesis equivalent - likely through link access
link = robot.get_link("gripperframe")
pos = link.get_pos()
```

### Setting Joint Targets

**MuJoCo:**
```python
data.ctrl[:] = joint_velocities
```

**Genesis:**
```python
robot.control_dofs_velocity(joint_velocities)
# or for position control:
robot.control_dofs_position(joint_positions)
```

### Contact Detection

**MuJoCo:**
```python
for i in range(data.ncon):
    contact = data.contact[i]
    geom1 = model.geom(contact.geom1).name
    geom2 = model.geom(contact.geom2).name
```

**Genesis:**
```python
# Need to investigate Genesis contact API
# Likely through collision detection callbacks or contact force queries
```

## Environment Porting Checklist

### Observation Space (21D)

| Component | MuJoCo Source | Genesis Equivalent |
|-----------|---------------|-------------------|
| Joint positions (6) | `data.qpos[joint_ids]` | `robot.get_dofs_position()` |
| Joint velocities (6) | `data.qvel[joint_ids]` | `robot.get_dofs_velocity()` |
| Gripper position (3) | `data.site_xpos[gripper_site]` | `robot.get_link("gripper").get_pos()` |
| Gripper orientation (3) | `Rotation.from_matrix(data.site_xmat[...]).as_euler('xyz')` | TBD |
| Cube position (3) | `data.xpos[cube_body_id]` | `cube.get_pos()` |

### Action Space (4D)

| Action | Description | Implementation |
|--------|-------------|----------------|
| dx | End-effector X delta | IK → joint velocities |
| dy | End-effector Y delta | IK → joint velocities |
| dz | End-effector Z delta | IK → joint velocities |
| gripper | Gripper open/close | Direct position control |

### Reward Components (v19)

All reward logic is portable - just need to extract the required state:

```python
# Required state for reward computation
gripper_pos = ...      # From Genesis
cube_pos = ...         # From Genesis
is_grasping = ...      # Contact detection
lift_height = 0.08     # Constant
```

### Curriculum Reset Stages

| Stage | Initial State | Notes |
|-------|--------------|-------|
| 0 | Normal (cube on table, arm at home) | Full task |
| 1 | Cube in closed gripper at lift height | Just hold |
| 2 | Cube in closed gripper on table | Need to lift |
| 3 | Gripper near cube, open | Need to grasp + lift |
| 4 | Gripper far from cube | Full pick task |

## IK Controller Porting

**MuJoCo IK** uses `mj_jac` for Jacobian computation:

```python
jacp = np.zeros((3, model.nv))
jacr = np.zeros((3, model.nv))
mujoco.mj_jac(model, data, jacp, jacr, point, body_id)
```

**Genesis options:**
1. Use Genesis built-in IK (if available)
2. Compute Jacobian manually using finite differences
3. Use differentiable physics for gradient-based IK

## Camera/Rendering Setup

**MuJoCo:**
```python
renderer = mujoco.Renderer(model, height=480, width=640)
renderer.update_scene(data, camera="wrist_cam")
image = renderer.render()
```

**Genesis (ray-traced):**
```python
# Add camera to scene
camera = scene.add_camera(
    res=(640, 480),
    pos=(x, y, z),
    lookat=(lx, ly, lz),
    fov=60,
)

# Render
image = camera.render(rgb=True)
```

## Training Integration

Genesis provides `OnPolicyRunner` for PPO training:

```python
from genesis.rl import OnPolicyRunner

runner = OnPolicyRunner(
    env=env,
    num_envs=8,
    num_steps=256,
    ...
)
runner.train(total_timesteps=1_000_000)
```

Alternatively, use custom training loop with PyTorch.

## Known Issues / TODOs

1. **MJCF Compatibility**: Some MuJoCo-specific features may not be supported
2. **Contact API**: Need to investigate Genesis contact detection
3. **Jacobian Access**: May need workaround for IK controller
4. **Multi-GPU**: Genesis supports DDP, explore for faster training

## Testing Strategy

1. **Unit test**: Load MJCF, verify joint/body structure
2. **Visual test**: Render scene, compare to MuJoCo
3. **Dynamics test**: Run scripted pick sequence, verify behavior
4. **Training test**: Run short training, verify learning signal
