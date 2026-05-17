# 011 — Grasping Physics Tuning

2026-05-17

## Problem

The SO-101 gripper in Isaac Sim can't reliably grasp the red cube. The cube slips out of the gripper during intuitive teleop grasping. It only works if the operator forces the cube between the fingers by pushing down before closing the gripper.

## Reference: MSSergeev/so101-lab

Found via LeRobot Discord. Repo at https://github.com/MSSergeev/so101-lab shows clean grasping in their README GIFs. Key differences from our setup identified below.

## Root Cause Analysis

Our sim uses default PhysX settings that are insufficient for stable small-object grasping:

1. **No `solve_articulation_contact_last`** — PhysX default solves articulation contacts first, which causes objects to "pop" out of articulated grippers
2. **Low solver iterations on the cube** — default 4 position iterations is too few for stable finger-object contact
3. **Default friction** — likely ~0.5 on cube, too slippery for the small contact area of the SO-101 gripper fingers
4. **No depenetration velocity cap** — when fingers contact the cube, PhysX can apply explosive separation forces

## Solution: Physics Settings from so101-lab

### 1. PhysX Scene Flag (most important)

```python
# Must be set on the PhysX scene prim
physx_scene.CreateSolveArticulationContactLastAttr(True)
```

Solves articulation contacts after all other contacts. Prevents objects from popping out of articulated grippers. Side effect: contact sensor force readings may be less accurate (irrelevant for teleop).

### 2. Graspable Object Solver Iterations

```python
# On the cube's rigid body prim
PhysxSchema.PhysxRigidBodyAPI:
    solver_position_iteration_count = 32   # was default 4
    solver_velocity_iteration_count = 4    # was default 1
    max_depenetration_velocity = 1.0       # was default unlimited
```

PhysX uses `max(objectA, objectB)` at each contact pair. Setting 32 on the cube means gripper-cube contacts get 32 iterations regardless of the robot's 8.

### 3. Friction on Graspable Objects

```python
# Physics material on the cube
static_friction = 0.8
dynamic_friction = 0.8
friction_combine_mode = "max"
```

`"max"` combine mode means effective friction = `max(cube_friction, gripper_friction)`, ensuring high friction even if one surface has low values.

### 4. Gripper Drive Settings

so101-lab uses:
```
stiffness = 10.8 Nm/rad
damping = 0.9 Nm·s/rad
effort_limit = 4 Nm
```

Our current values (MuammerBay):
```
stiffness = 400.0
damping = 40.0
maxForce = 100.0
```

Our values are much higher, which may cause the gripper to crush through objects or create unstable contacts. The so101-lab values are for Isaac Lab's implicit actuator model which works differently from direct USD drives, so we need to be careful about direct comparison. The key insight is that lower, softer drives give more stable grasps because PhysX has time to resolve contacts.

### 5. Robot Articulation Settings

so101-lab uses:
```
solver_position_iteration_count = 8   # same as ours
solver_velocity_iteration_count = 4   # ours is 0
enabled_self_collisions = True        # same as ours
```

Adding velocity iterations (4 vs our 0) may help with contact stability.

## Settings Applied

Changes made to `sim_teleop.py`:

| Setting | Before | After | Where |
|---------|--------|-------|-------|
| `solveArticulationContactLast` | not set (false) | true | PhysX scene prim |
| Cube solver position iters | default (4) | 32 | GraspCube rigid body |
| Cube solver velocity iters | default (1) | 4 | GraspCube rigid body |
| Cube max depenetration vel | default (unlimited) | 1.0 | GraspCube rigid body |
| Cube static friction | default (~0.5) | 0.8 | GraspCube physics material |
| Cube dynamic friction | default (~0.5) | 0.8 | GraspCube physics material |
| Cube friction combine mode | default (average) | max | GraspCube physics material |
| Robot velocity solver iters | 0 | 4 | Robot articulation |
| Gripper stiffness | 400.0 | 50.0 | Gripper drive (reduced) |
| Gripper damping | 40.0 | 10.0 | Gripper drive (reduced) |
| Gripper maxForce | 100.0 | 5.0 | Gripper drive (reduced) |

Gripper drive values are a compromise between our original (too stiff) and so101-lab's Isaac Lab values (different actuator model). These are softer than before but still use direct USD drives.

## Files

| File | Change |
|------|--------|
| `scripts/sim_teleop.py` | Added PhysX scene flag, cube physics, gripper drive tuning |

## References

- https://github.com/MSSergeev/so101-lab — SO-101 Isaac Lab environment with clean grasping
- `so101_lab/assets/robots/so101.py` — actuator configs
- `so101_lab/tasks/figure_shape_placement/env_cfg.py` — object physics settings
- `docs/physics_tuning.md` in that repo — explains `solve_articulation_contact_last`
