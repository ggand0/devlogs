# GPU Dust Ring Implementation

**Date**: December 23, 2025
**Status**: COMPLETE

## Overview

Migrated the CPU dust ring emitter to GPU using bevy_hanabi. This was initially flagged as "low priority" due to only 2-3 particles, but proved straightforward with existing modifiers.

## CPU Behavior (Reference)

From `spawn_dust_ring` in ground_explosion.rs:

| Property | Value |
|----------|-------|
| Count | 2-3 particles |
| Lifetime | 0.1-0.5s (very short) |
| Spawn | Exact origin |
| Velocity | 35° cone upward, 5-10 m/s |
| Size | 3-5m base, scales from ZERO to (3×, 2×) |
| Scale curve | Non-uniform: X grows to 3×, Y grows to 2× |
| Color | Dark brown (0.147, 0.114, 0.070) |
| Alpha | 3.0→0 S-curve fade (HDR brightness) |
| Texture | 4×1 flipbook, random FIXED frame |
| Orientation | VelocityAligned |

## Key Insight: No Custom Modifiers Needed

Initial evaluation suggested needing custom modifiers for:
1. Non-uniform X/Y scaling
2. Random fixed frame selection

However, bevy_hanabi's existing modifiers handle both:

### Non-Uniform Scaling

`SizeOverLifetimeModifier` accepts `Gradient<Vec3>` - each key can have different X, Y, Z values:

```rust
let mut dust_size_gradient = bevy_hanabi::Gradient::new();
dust_size_gradient.add_key(0.0, Vec3::new(0.0, 0.0, 1.0));    // Start at zero
dust_size_gradient.add_key(1.0, Vec3::new(8.0, 12.0, 1.0));   // End at 2×, 3× (swapped for GPU)
```

Note: bevy_hanabi's `AlongVelocity` orientation swaps X/Y compared to CPU:
- CPU: Y = along velocity (height), X = perpendicular (width)
- GPU: X = along velocity, Y = perpendicular

So we swap: GPU_X = CPU_Y (2×), GPU_Y = CPU_X (3×)

### Random Fixed Frame

`FlipbookModifier` reads `Attribute::SPRITE_INDEX`. For a fixed random frame, simply set it at init and never update:

```rust
// Cast rand float to int: floor(rand * 4) gives 0, 1, 2, or 3
let dust_random_frame = (writer_dust.rand(ScalarType::Float) * writer_dust.lit(4.0))
    .cast(ScalarType::Int);
let dust_init_sprite = SetAttributeModifier::new(Attribute::SPRITE_INDEX, dust_random_frame.expr());
```

Combined with `FlipbookModifier { sprite_grid_size: UVec2::new(4, 1) }`, this picks one random frame at spawn.

## Implementation

### Color Gradient (HDR Alpha)

S-curve fade from 3.0 to 0.0 (UE5 uses alpha > 1.0 as brightness multiplier):

```rust
let mut dust_color_gradient = bevy_hanabi::Gradient::new();
// S-curve: 3.0 * (1 - smoothstep(t))
dust_color_gradient.add_key(0.0, Vec4::new(0.147, 0.114, 0.070, 3.0));
dust_color_gradient.add_key(0.2, Vec4::new(0.147, 0.114, 0.070, 2.59));
dust_color_gradient.add_key(0.5, Vec4::new(0.147, 0.114, 0.070, 1.5));
dust_color_gradient.add_key(0.8, Vec4::new(0.147, 0.114, 0.070, 0.34));
dust_color_gradient.add_key(1.0, Vec4::new(0.147, 0.114, 0.070, 0.0));
```

### Velocity (35° Cone)

```rust
let dust_theta = writer_dust.rand(ScalarType::Float) * writer_dust.lit(TAU);
let dust_cone_angle = writer_dust.lit(35.0_f32.to_radians());
let dust_phi = writer_dust.rand(ScalarType::Float) * dust_cone_angle;
let dust_dir = vec3(sin(phi) * cos(theta), cos(phi), sin(phi) * sin(theta));
let dust_speed = 5.0 + rand() * 5.0; // 5-10 m/s
```

### Effect Definition

```rust
let ground_dust_effect = effects.add(
    EffectAsset::new(8, SpawnerSettings::once(3.0.into()), dust_module)
        .with_name("ground_explosion_dust_ring")
        .with_alpha_mode(bevy_hanabi::AlphaMode::Blend)
        .init(dust_init_pos)
        .init(dust_init_vel)
        .init(dust_init_age)
        .init(dust_init_lifetime)
        .init(dust_init_sprite)
        .render(OrientModifier::new(OrientMode::AlongVelocity))
        .render(ParticleTextureModifier { ... })
        .render(FlipbookModifier { sprite_grid_size: UVec2::new(4, 1) })
        .render(ColorOverLifetimeModifier::new(dust_color_gradient))
        .render(SizeOverLifetimeModifier { gradient: dust_size_gradient, ... })
);
```

## Key Bindings

| Key | Effect |
|-----|--------|
| 5 | CPU dust ring (2-3 entities) |
| Shift+5 | GPU dust ring (1 entity) |
| J | Full GPU explosion + dust + impact |

## Files Changed

- `src/particles.rs`: Added GPU dust effect, texture loading, spawn function
- `src/ground_explosion.rs`: Added Shift+5 handler, updated J key to include GPU dust

## GPU Emitter Status Update

| Emitter | CPU Entities | GPU | Status |
|---------|-------------|-----|--------|
| Main Fireball | 9-17 | 1 | ✅ Done |
| Secondary Fireball | 7-13 | 1 | ✅ Done |
| Sparks | 30-60 | 1 | ✅ Done |
| Flash Sparks | 20-50 | 1 | ✅ Done |
| Parts Debris | 50-75 | 1 | ✅ Done |
| Dirt Debris | ~35 | 1 | ✅ Done |
| Velocity Dirt | 10-15 | 1 | ✅ Done |
| **Dust Ring** | 2-3 | 1 | ✅ **Done** |
| Smoke Cloud | ~10-20 | - | CPU only |
| Wisp Puffs | ~15-25 | - | CPU only |
| Impact Flash | 1 | - | CPU only (low priority) |

**8 of 11 emitters** now have GPU implementations.
