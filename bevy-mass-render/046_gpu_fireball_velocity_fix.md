# GPU Fireball Velocity Fix

**Date**: December 22, 2025
**Status**: RESOLVED

## Problem

GPU fireball particles appeared to move INWARD toward the spawn center instead of OUTWARD. This was documented as a velocity direction issue in the handoff (045).

## Root Cause

Two separate issues:

### 1. UVScaleOverLifetimeModifier Bug (bevy_hanabi fork)

The custom `UVScaleOverLifetimeModifier` had incorrect math:

```wgsl
// WRONG: multiplying by scale sends UVs out of bounds
let uv_scaled = uv_centered * uv_scale + vec2<f32>(0.5, 0.5);
// With scale=500: UV 0.0 becomes -249.5, UV 1.0 becomes 250.5
```

This caused textures to sample edge pixels (solid color) instead of zooming in on the center. The "zoom out" effect created a visual illusion of inward movement.

**Fix**: Divide instead of multiply:

```wgsl
// CORRECT: dividing by scale zooms IN on center
let uv_scaled = uv_centered / uv_scale + vec2<f32>(0.5, 0.5);
// With scale=500: UV 0.0 becomes 0.499, UV 1.0 becomes 0.501 (tiny region around center)
```

### 2. Velocity Direction (bevy-mass-render)

The velocity setup now uses `writer.attr(Attribute::POSITION)` to read the already-set position:

```rust
// Position: hemisphere surface
let fb_pos = rx.vec3(ry, rz).normalized() * writer_fireball.lit(0.5);
let fireball_init_pos = SetAttributeModifier::new(Attribute::POSITION, fb_pos.expr());

// Velocity: read position back, normalize, multiply by speed
let fb_pos_read = writer_fireball.attr(Attribute::POSITION);
let fb_outward_dir = fb_pos_read.normalized();
let fb_velocity = fb_outward_dir * fb_speed;
let fireball_init_vel = SetAttributeModifier::new(Attribute::VELOCITY, fb_velocity.expr());
```

This works because `attr(POSITION)` in the expression system reads the particle's position field after it's been set by the position modifier.

## What Was NOT the Problem

- `SetVelocitySphereModifier` - Actually works correctly, but wasn't the issue
- Expression re-evaluation with `clone()` - Not relevant when using `attr()` to read back
- Init modifier ordering - Ordering is correct, code appends sequentially

## Debug Tools Added

- `debug_fireball_effect` - Small velocity-colored quads (X key)
- Colors show velocity direction: R=+X, G=+Y, B=+Z
- Useful for verifying actual particle movement vs visual perception

## Commits

1. bevy_hanabi (custom-mods branch):
   - `1ca25cb` - Fix UVScaleOverLifetimeModifier: divide instead of multiply

2. bevy-mass-render (feat/gpu-particle-perf branch):
   - `ed1781f` - GPU fireball: fix outward velocity with attr(POSITION) approach

## Key Insight

When debugging particle effects, separate the actual physics (position/velocity) from visual rendering (textures, UV effects, orientation). The debug effect with simple colored quads immediately showed particles were actually moving correctly - the visual "inward" appearance was purely from the broken UV zoom effect.
