# GPU Fireball Velocity Debug Investigation

## Problem

GPU fireballs appear to move "inward toward center then fly out the other side" instead of scattering outward immediately from spawn position. User reported velocity looks "mirrored against some plane."

## Empirical Testing Methodology

Created debug tests with fixed positions and velocities to isolate the issue.

### Test 1: Fixed Position + Fixed Velocity

```rust
// Position (5, 2, 0), Velocity (10, 0, 0)
// Expected: move RIGHT
```
**Result**: Particle moves RIGHT - correct.

### Test 2: Fixed Position + SetVelocitySphereModifier

```rust
// Position (5, 2, 0) fixed
// SetVelocitySphereModifier { center: ZERO, speed: 10.0 }
// Expected: velocity = normalize((5,2,0)) * 10 = move RIGHT+UP
```
**Result**: Particle moves RIGHT+UP - correct.

### Test 3: SetPositionSphereModifier + SetVelocitySphereModifier

```rust
// SetPositionSphereModifier { center: ZERO, radius: 3.0, dimension: Surface }
// SetVelocitySphereModifier { center: ZERO, speed: 10.0 }
// Expected: particles expand outward from center
```
**Result**: Particles expand OUTWARD - correct with `ParallelCameraDepthPlane` orientation.

### Test 4: Same but with AlongVelocity orientation

```rust
.render(OrientModifier::new(OrientMode::AlongVelocity))
```
**Result**: Particles expand outward but look like "spirals" - the AlongVelocity orientation stretches quads along velocity direction, creating distorted appearance.

## Key Finding

**The velocity direction is CORRECT.** The visual "mirroring" issue is NOT a velocity problem - it's caused by:

1. **AlongVelocity orientation** - stretches particle quads along velocity, making spherical scatter look like spirals
2. **-90° rotation** - required to match CPU's "up axis = velocity" orientation, but combined with sphere spawn creates unexpected visual
3. **Axis rotation (random spin)** - adds additional visual complexity

## OrientMode Comparison

| Mode | X Axis | Y Axis | Z Axis |
|------|--------|--------|--------|
| bevy_hanabi AlongVelocity | Along velocity | Perpendicular | Toward camera |
| CPU VelocityAligned | Perpendicular | Along velocity (up) | Toward camera |

CPU requires -90° rotation to convert between these coordinate systems.

## SimulationSpace Findings

### Global Space (default)
- Particles in world space
- Transform.scale IGNORED
- Velocity in world units

### Local Space
- Particles relative to effect transform
- Transform.scale APPLIED to positions AND velocities
- Effect at scale 2.0 = particles move 2x faster in world

## Test Matrix Summary

| SimSpace | OrientMode | Rotation | Result |
|----------|------------|----------|--------|
| Global | ParallelCameraDepthPlane | None | Outward scatter, billboards face camera |
| Global | AlongVelocity | None | Outward spiral (stretched quads) |
| Global | AlongVelocity | -90° + spin | Same issue as reported (visual mirroring) |
| Local | ParallelCameraDepthPlane | None | Outward scatter, billboards face camera |
| Local | AlongVelocity | -90° + spin | Same issue as reported (visual mirroring) |

## Hypothesis

The "mirroring" visual effect occurs because:

1. Particles spawn on a sphere surface in random directions
2. AlongVelocity orients each quad along its velocity vector
3. The -90° rotation makes Y axis point along velocity (like CPU)
4. But the fireball texture is designed for a specific orientation
5. When velocity points in certain directions relative to camera, the texture appears "backwards"

This is NOT a velocity direction bug - it's a texture/orientation mismatch when particles move in directions not anticipated by the texture design.

## Current Status

- Sphere position modifier: Working correctly
- Sphere velocity modifier: Working correctly
- Billboard orientation: Works, but doesn't match CPU look
- AlongVelocity orientation: Creates visual artifacts with radial scatter

## Next Steps

Options to consider:
1. Use billboard orientation and accept difference from CPU
2. Investigate why CPU VelocityAligned looks correct but GPU AlongVelocity doesn't
3. Consider if hemisphere spawn (upper only) would help
4. Check if the fireball texture requires specific orientation assumptions
