# GPU Fireball Horizontal Mirroring Fix

## Problem

GPU fireballs with `AlongVelocity` orientation and `-90° rotation` showed horizontal texture mirroring when particles had opposite velocity directions. This made the explosion look like particles were "moving inward toward center then flying out the other side" visually, even though the actual velocity was correct.

## Root Cause Analysis

### The Cross Product Sign Flip

In the original AlongVelocity code (`bevy_hanabi/src/modifier/output.rs`):

```wgsl
let dir = normalize(position - get_camera_position_effect_space());  // Away from camera
let axis_x0 = normalize(particle.velocity);
let axis_y0 = cross(dir, axis_x0);  // THIS FLIPS WHEN VELOCITY FLIPS
```

When `velocity` points UP (0, 1, 0):
- `axis_y0 = cross(dir, (0,1,0)) = (1, 0, 0)`  (pointing right)

When `velocity` points DOWN (0, -1, 0):
- `axis_y0 = cross(dir, (0,-1,0)) = (-1, 0, 0)`  (pointing left)

### The -90° Rotation Amplifies the Problem

After the -90° rotation (cos=0, sin=-1):
```wgsl
axis_x = axis_x0 * 0 + axis_y0 * (-1) = -axis_y0
axis_y = axis_y0 * 0 - axis_x0 * (-1) = axis_x0
```

So:
- Velocity UP: `axis_x = -(1,0,0) = (-1,0,0)`, `axis_y = (0,1,0)`
- Velocity DOWN: `axis_x = -(-1,0,0) = (1,0,0)`, `axis_y = (0,-1,0)`

The texture's **horizontal axis (axis_x)** is different! This causes horizontal mirroring.

### Why CPU Doesn't Have This Issue

CPU uses both `right` and `up` which flip together:
```rust
let up = velocity_dir;
let right = up.cross(forward).normalize();
```

When both right AND up flip, the quad rotates 180° (not mirrored). The GPU's double-negation through `-axis_y0` breaks this symmetry.

## The Fix

Force `axis_y0` to always point in a consistent world-space direction regardless of velocity direction, by checking against a reference vector.

```wgsl
let to_camera = normalize(get_camera_position_effect_space() - position);
let axis_x0 = normalize(particle.velocity);
let raw_axis_y0 = cross(-to_camera, axis_x0);

// Force axis_y0 to consistent direction using world-right reference
let world_right = cross(vec3<f32>(0.0, 1.0, 0.0), to_camera);
let y0_sign = sign(dot(raw_axis_y0, world_right) + 0.0001);
let axis_y0 = raw_axis_y0 * y0_sign;
```

This ensures `axis_y0` always aligns with "world right relative to camera" (positive dot product), so particles moving up and down both render with consistent texture orientation.

## Verification

CPU readback confirmed particle physics is correct:
```
Particle 0: pos=(-0.386, -3.409, -0.753), vel=(-0.549, -4.853, -1.072)
Particle 3: pos=(3.151, 1.548, 0.092), vel=(4.486, 2.204, 0.131)
```

Both particles have position and velocity pointing in the same direction (outward from center), confirming the physics was never broken - only the visual orientation.

## Files Modified

- `/home/gota/ggando/gamedev/bevy_hanabi/src/modifier/output.rs` (lines 657-693)
  - `OrientMode::AlongVelocity` branch
  - Both the "with rotation" and "without rotation" cases

## Test Configuration

- `SetPositionSphereModifier` - spawn on sphere surface
- `SetVelocitySphereModifier` - velocity radially outward
- `OrientMode::AlongVelocity` with `-90° rotation`
- 3 particles for clear observation
