# MuJoCo Contact Slip Issue and Fix

## Problem

When grasping a wooden cube (friction=0.5) in MuJoCo, the cube slips out of the gripper during lift - even with tight grip force. This doesn't happen on the real SO-101 robot with the same grasp.

## Two Types of Slippage

From [MuJoCo Modeling - Contact Slippage](https://mujoco.readthedocs.io/en/latest/modeling.html#cslippage):

### 1. Fast Slippage
Occurs when "slip-preventing contact forces are outside the friction cone" - the physics cannot prevent slipping regardless of configuration.

**Fixes:**
- Increase sliding friction coefficient
- Increase normal force (better gripper design)
- Improve contact geometry with multiple contact points
- Enable `multiccd` flag for more contact detection
- Enable `nativeccd` flag for better collision detection
- Set `condim=4` for torsional friction or `condim=6` for rolling friction

### 2. Slow Slippage
"A property of MuJoCo's contact model by design, since without it the inverse dynamics are not defined."

From [MuJoCo Overview - Softness and Slip](https://mujoco.readthedocs.io/en/latest/overview.html#softness-and-slip):

> "Gradual contact slip cannot be avoided... This is not because the solver is unable to prevent slip, but because it is not trying to prevent slip in the first place."

The soft contact model means pushing harder always produces larger acceleration - if slip were completely eliminated, this core property would be violated.

**Fixes:**
- Increase `impratio` parameter (works best with elliptic cones)
- Enable noslip solver with positive iterations (1-3)

## Recommended Solution

From the docs: **"The Newton solver with elliptic friction cones and large value of impratio is the recommended way of reducing slip."**

This provides better accuracy than the `noslip` solver, which can reduce robustness by abandoning MuJoCo's convex optimization formulation.

## The Fix Applied

### Before (slipping)
```xml
<option timestep="0.002" gravity="0 0 -9.81" noslip_iterations="3"/>
```

### After (stable grasp)
```xml
<option timestep="0.002" gravity="0 0 -9.81" noslip_iterations="3" impratio="10" cone="elliptic"/>
```

### Parameters Explained

| Parameter | Value | Effect |
|-----------|-------|--------|
| `impratio` | 10 | Friction forces 10x "harder" than normal forces |
| `cone` | elliptic | Required for `impratio` to work effectively |
| `noslip_iterations` | 3 | Additional no-slip solver iterations (backup) |

**Note:** High `impratio` values may require smaller timesteps for stability since nonlinear dynamics become harder to integrate.

## Solver Parameters

### `solref` - Reference Behavior
Controls the spring-damper dynamics for constraint satisfaction.
- Format: `[timeconst, dampratio]`
- Example: `solref="0.001 1"` = very stiff (0.001s time constant)

### `solimp` - Impedance
Controls the solver's ability to prevent constraint violation.
- Format: `[dmin, dmax, width]`
- Increasing first two elements reduces slip
- Example: `solimp="0.99 0.99 0.001"` = high impedance

### `impratio` - Friction/Normal Impedance Ratio
Ratio of frictional-to-normal constraint impedance for elliptic friction cones.
- Values >1 make friction "harder" than normal forces
- Prevents slip without increasing friction coefficient
- Works only with `cone="elliptic"`

## Why Real Robot Doesn't Slip

1. **Rubber/textured pads** on real gripper create better friction than smooth PLA mesh
2. **Deformable contact** - real materials deform, increasing contact area
3. **No soft contact model** - real physics doesn't have solver-induced slip
4. **Multiple contact points** - real finger surfaces create distributed contacts

## Additional Improvements

For robust grasping, also consider:

1. **Better collision geometry** - add invisible box geoms at fingertips (mesh collisions less stable)
2. **Enable multiccd** - `<flag multiccd="enable"/>` for more contact points
3. **Increase condim** - set `condim="4"` on gripper geoms for torsional friction

## Current Scene Configuration

```xml
<!-- lift_cube_scene.xml -->
<option timestep="0.002" gravity="0 0 -9.81" noslip_iterations="3" impratio="10" cone="elliptic"/>

<default>
    <geom solref="0.001 1" solimp="0.99 0.99 0.001"/>
</default>

<!-- Cube with wood-like friction -->
<geom name="cube_geom" type="box" size="0.015 0.015 0.015"
      mass="0.03" friction="0.5 0.05 0.001"/>
```

## Friction Reference Values

From [Engineering Toolbox](https://www.engineeringtoolbox.com/friction-coefficients-d_778.html):

| Material Pair | Static Friction |
|--------------|-----------------|
| Wood on wood (dry) | 0.25-0.5 |
| Wood on metal | 0.2-0.6 |
| Rubber on concrete | 0.6-0.85 |
| Rubber on rubber | ~1.16 |

## Key Insights from GitHub Issues

### Issue #239 - Object slipping from end-effector
- MuJoCo developer confirms: "slip is a feature, not a bug" - required for well-defined inverse dynamics
- Recommended fix: `impratio=10` with `cone="elliptic"`
- Alternative: `noslip_iterations` but less robust than impratio approach

### Discussion #656 - impratio explanation
- `impratio` controls ratio of friction impedance to normal impedance
- Default is 1.0 (equal). Values >1 make friction "stiffer"
- Only works with elliptic cones, not pyramidal
- MuJoCo's official gripper example uses `impratio="10"`

## Sim2Real Transfer

### Domain Randomization
Key technique used by OpenAI for Rubik's Cube sim2real:
- Randomize friction coefficients during training
- Randomize object mass and inertia
- Randomize actuator gains
- Agent learns robust policy that transfers to real hardware

### MuJoCo Sim2Real Production Examples

- **Covariant** - RL-trained warehouse pick-and-place robots deployed at Gap, Crate & Barrel. Founded by OpenAI robotics team (Pieter Abbeel, Rocky Duan, Peter Chen).
- **OpenAI Rubik's Cube (2019)** - Shadow Dexterous Hand trained in MuJoCo, solved physical Rubik's Cube using Automatic Domain Randomization (ADR).

## Alternative Simulators

- **Isaac Sim / Isaac Lab** - NVIDIA GPU-accelerated simulator, thousands of parallel envs. Requires NVIDIA GPU.
- **MuJoCo** - CPU-based, simpler setup, well-documented contact model.

## Remaining Issues

With current settings (`impratio=10`, elliptic cones), grasp works but exhibits slight finger penetration and wobble during lift.

### Potential Fix: multiccd

From [MuJoCo XML Reference](https://mujoco.readthedocs.io/en/stable/XMLreference.html):

> `multiccd: [disable, enable], "disable"`
> This flag enables multiple-contact collision detection for geom pairs that use a general-purpose convex-convex collider e.g., mesh-mesh collisions. This can be useful when the contacting geoms have a flat surface and the single contact point generated by the convex-convex collider cannot accurately capture the surface contact, leading to instabilities that typically manifest as **sliding or wobbling**.

To enable:
```xml
<flag multiccd="enable"/>
```

This generates multiple contact points for mesh-mesh collisions instead of a single point, which should stabilize flat surface contacts.

## Sources

### MuJoCo Documentation
- [MuJoCo Overview - Softness and Slip](https://mujoco.readthedocs.io/en/latest/overview.html#softness-and-slip)
- [MuJoCo Modeling - Contact Slippage](https://mujoco.readthedocs.io/en/latest/modeling.html#cslippage)
- [MuJoCo XML Reference - multiccd flag](https://mujoco.readthedocs.io/en/stable/XMLreference.html) - Multiple contact points for mesh collisions
- [MuJoCo Playground - Gripper Example](https://playground.mujoco.org/) - Uses `cone="elliptic" impratio="10"`

### GitHub Issues
- [Issue #239 - Object slipping from end-effector](https://github.com/google-deepmind/mujoco/issues/239)
- [Discussion #656 - impratio explanation](https://github.com/google-deepmind/mujoco/discussions/656)

### Sim2Real Examples
- [Covariant AI](https://covariant.ai/) - Production warehouse RL
- [OpenAI Rubik's Cube](https://openai.com/index/solving-rubiks-cube/) - MuJoCo sim2real with ADR
- [MuJoCo Menagerie](https://github.com/google-deepmind/mujoco_menagerie) - Collection of robot models

### Alternative Simulators
- [Isaac Lab](https://isaac-sim.github.io/IsaacLab/) - NVIDIA GPU-accelerated

### Reference
- [Engineering Toolbox - Friction Coefficients](https://www.engineeringtoolbox.com/friction-coefficients-d_778.html)
