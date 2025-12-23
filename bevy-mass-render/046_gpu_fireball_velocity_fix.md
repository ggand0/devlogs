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

---

## Visual Tuning (Post-Fix)

After fixing the UV zoom bug, additional tuning was needed to match the CPU fireball appearance.

### Spawn Hemisphere Size

The initial 0.5m radius (1m diameter) was too small - particles appeared stationary because the large particle sizes (8-20m) completely obscured the outward movement.

**Solution**: Increased spawn hemisphere to 7.5m radius (15m diameter):

```rust
// Position: hemisphere (Y >= 0) surface, radius 7.5 (15m diameter)
let rx = writer_fireball.rand(ScalarType::Float) * writer_fireball.lit(2.0) - writer_fireball.lit(1.0);
let ry = writer_fireball.rand(ScalarType::Float); // [0,1] for Y >= 0
let rz = writer_fireball.rand(ScalarType::Float) * writer_fireball.lit(2.0) - writer_fireball.lit(1.0);
let fb_pos = rx.vec3(ry, rz).normalized() * writer_fireball.lit(7.5);
```

### Size Gradient: Cubic Ease-Out Curve

The CPU fireball uses a cubic ease-out curve (`1 - (1-t)³`) for size scaling from 0.5× to 1.3× over lifetime. This creates fast initial expansion that slows down.

**Cubic ease-out formula**: `scale = 0.5 + 0.8 * (1 - (1-t)³)`

| t | (1-t)³ | 1-(1-t)³ | scale |
|---|--------|----------|-------|
| 0.0 | 1.0 | 0.0 | 0.5× |
| 0.2 | 0.512 | 0.488 | 0.89× |
| 0.4 | 0.216 | 0.784 | 1.127× |
| 0.6 | 0.064 | 0.936 | 1.249× |
| 0.8 | 0.008 | 0.992 | 1.294× |
| 1.0 | 0.0 | 1.0 | 1.3× |

**GPU implementation** using gradient keys (base size 16m):

```rust
let mut fireball_size_gradient = bevy_hanabi::Gradient::new();
fireball_size_gradient.add_key(0.0, Vec3::splat(8.0));    // 0.5× = 8m
fireball_size_gradient.add_key(0.2, Vec3::splat(14.2));   // 0.89× = 14.2m
fireball_size_gradient.add_key(0.4, Vec3::splat(18.0));   // 1.127× = 18m
fireball_size_gradient.add_key(0.6, Vec3::splat(20.0));   // 1.249× = 20m
fireball_size_gradient.add_key(0.8, Vec3::splat(20.7));   // 1.294× = 20.7m
fireball_size_gradient.add_key(1.0, Vec3::splat(21.0));   // 1.3× = 21m
```

### UV Zoom Gradient

The UV zoom creates the "emerging from center" effect. Scale starts at 500 (extremely zoomed in) and eases out to 1 (full texture visible).

```rust
.render(UVScaleOverLifetimeModifier {
    gradient: {
        let mut g = bevy_hanabi::Gradient::new();
        g.add_key(0.0, Vec2::splat(500.0));  // Tiny center region
        g.add_key(0.2, Vec2::splat(466.0));  // Fast initial zoom-out
        g.add_key(0.4, Vec2::splat(350.0));
        g.add_key(0.6, Vec2::splat(224.0));
        g.add_key(0.8, Vec2::splat(100.0));
        g.add_key(1.0, Vec2::splat(1.0));    // Full texture
        g
    },
})
```

### Velocity Tuning

Increased outward velocity from 3-5 m/s to 5-8 m/s to match CPU scatter distance:

```rust
let fb_speed = writer_fireball.lit(5.0)
    + writer_fireball.rand(ScalarType::Float) * writer_fireball.lit(3.0); // 5-8 m/s
```

### Modifier Order

**Critical**: UV zoom must run BEFORE FlipbookModifier. The CPU shader applies UV zoom to base UVs, then the flipbook offset is added. If reversed, the zoom centers on the wrong point.

```rust
// CORRECT ORDER:
.render(UVScaleOverLifetimeModifier { ... })  // First: zoom base UVs
.render(FlipbookModifier { sprite_grid_size: UVec2::new(8, 8) })  // Then: offset to frame
```

## Final Result

The GPU fireball now visually matches the CPU version:
- Particles spawn on 15m diameter hemisphere
- Fast initial size expansion (cubic ease-out)
- UV zoom creates "emerging from center" effect
- Outward velocity 5-8 m/s
- J key: GPU explosion + impact flash (clean test)
- K key: Full explosion with all emitters
