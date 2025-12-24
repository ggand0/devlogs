# GPU Smoke Cloud & Wisp Puffs Implementation

**Date**: December 23, 2025
**Status**: COMPLETE

## Overview

Migrated the remaining two CPU emitters (smoke cloud and wisp puffs) to GPU using bevy_hanabi. With these additions, 10 of 11 ground explosion emitters now have GPU implementations.

## GPU Smoke Cloud

### CPU Behavior (Reference)

From `spawn_smoke_cloud` in ground_explosion.rs:

| Property | Value |
|----------|-------|
| Count | 10-15 particles |
| Lifetime | 0.8-2.5s |
| Spawn | Origin with slight Y offset |
| Velocity | Screen-local spread ±12m/s (XY), minimal Z |
| Size | 0.5-1.0m base, grows 3× over lifetime |
| Color | Grey (0.4, 0.4, 0.4), alpha 0.6→0 |
| Texture | 8×8 flipbook, 35 frames animated |
| Orientation | Camera facing |
| Physics | Drag 2.0, upward accel 0.5 m/s² |

### GPU Implementation

```rust
// Color: constant grey with alpha fade 0.6→0
let mut smoke_color_gradient = bevy_hanabi::Gradient::new();
smoke_color_gradient.add_key(0.0, Vec4::new(0.4, 0.4, 0.4, 0.6));
smoke_color_gradient.add_key(0.5, Vec4::new(0.4, 0.4, 0.4, 0.4));
smoke_color_gradient.add_key(1.0, Vec4::new(0.4, 0.4, 0.4, 0.0));

// Size: grow from 0.75m to 2.25m (3×)
let mut smoke_size_gradient = bevy_hanabi::Gradient::new();
smoke_size_gradient.add_key(0.0, Vec3::splat(0.75));
smoke_size_gradient.add_key(1.0, Vec3::splat(2.25));

// Velocity: XZ spread (world-space equivalent of screen-local)
let smoke_vel_x = (rand() * 24.0) - 12.0;  // ±12 m/s
let smoke_vel_y = rand() * 2.0;             // Slight upward
let smoke_vel_z = (rand() * 24.0) - 12.0;  // ±12 m/s

// Physics modifiers
.update(LinearDragModifier::new(2.0))
.update(AccelModifier::new(Vec3::new(0.0, 0.5, 0.0)))
```

### Screen-Local vs World-Space Velocity

The CPU smoke uses `bLocalSpace=True` which spreads velocity on the screen plane (camera XY). For GPU, we use world XZ spread which produces a similar visual result when viewed from typical camera angles.

## GPU Wisp Puffs

### CPU Behavior (Reference)

From `spawn_wisps` in ground_explosion.rs:

| Property | Value |
|----------|-------|
| Count | 3 particles |
| Lifetime | 1.0-2.0s |
| Spawn | Origin + 0.5m Y offset |
| Velocity | Upward 3-6 m/s, horizontal ±1 m/s |
| Size | 0→5× cubic ease-in, 0.8-1.8m base (×1.5 scale) |
| Color | Dark grey (0.15, 0.12, 0.10), alpha 3.0→0 HDR |
| Texture | 8×8 flipbook, 64 frames animated |
| Orientation | Camera facing |
| Physics | Gravity 9.8 m/s² |

### GPU Implementation

```rust
// Color: dark grey with HDR alpha fade (3.0→0)
let mut wisp_color_gradient = bevy_hanabi::Gradient::new();
wisp_color_gradient.add_key(0.0, Vec4::new(0.15, 0.12, 0.10, 3.0));
wisp_color_gradient.add_key(0.1, Vec4::new(0.15, 0.12, 0.10, 2.7));
wisp_color_gradient.add_key(0.5, Vec4::new(0.15, 0.12, 0.10, 1.5));
wisp_color_gradient.add_key(1.0, Vec4::new(0.15, 0.12, 0.10, 0.0));

// Size: cubic ease-in (t³) from 0 to 10m
let mut wisp_size_gradient = bevy_hanabi::Gradient::new();
wisp_size_gradient.add_key(0.0, Vec3::splat(0.0));     // Start at zero
wisp_size_gradient.add_key(0.2, Vec3::splat(0.16));    // 0.2³ × 10 × 2
wisp_size_gradient.add_key(0.4, Vec3::splat(1.28));    // 0.4³ × 10 × 2
wisp_size_gradient.add_key(0.6, Vec3::splat(4.32));    // 0.6³ × 10 × 2
wisp_size_gradient.add_key(0.8, Vec3::splat(10.24));   // 0.8³ × 10 × 2
wisp_size_gradient.add_key(1.0, Vec3::splat(10.0));    // 5× final

// Velocity: gentle upward with horizontal spread (×1.5 scale modifier)
let wisp_vel_x = (rand() * 3.0) - 1.5;   // ±1.5 m/s
let wisp_vel_y = 4.5 + rand() * 4.5;     // 4.5-9 m/s (3-6 × 1.5)
let wisp_vel_z = (rand() * 3.0) - 1.5;   // ±1.5 m/s

// Gravity
.update(AccelModifier::new(Vec3::new(0.0, -9.8, 0.0)))
```

### Cubic Ease-In Scale Curve

The CPU uses `t³` for scale growth (slow start, fast finish). Gradient approximation:

| t | t³ | scale (2m base × 5×) |
|---|-----|---------------------|
| 0.0 | 0.000 | 0.0m |
| 0.2 | 0.008 | 0.16m |
| 0.4 | 0.064 | 1.28m |
| 0.6 | 0.216 | 4.32m |
| 0.8 | 0.512 | 10.24m |
| 1.0 | 1.000 | 10.0m |

## Key Bindings

| Key | Effect |
|-----|--------|
| 6 | CPU wisp (3 entities) |
| Shift+6 | GPU wisp (1 entity) |
| 7 | CPU smoke (10-15 entities) |
| Shift+7 | GPU smoke (1 entity) |
| J | FULL GPU explosion (all emitters) |
| K | Full CPU/GPU mix explosion |

## Files Changed

- `src/particles.rs`: Added GPU smoke/wisp effects, textures, spawn functions
- `src/ground_explosion.rs`: Added Shift+6/7 handlers, updated J key to full GPU

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
| Dust Ring | 2-3 | 1 | ✅ Done |
| **Smoke Cloud** | 10-15 | 1 | ✅ **Done** |
| **Wisp Puffs** | 3 | 1 | ✅ **Done** |
| Impact Flash | 1 | - | CPU only |

**10 of 11 emitters** now have GPU implementations.

## Entity Count Comparison

| Mode | Entities per Explosion |
|------|----------------------|
| Full CPU | ~170-290 entities |
| Full GPU (J key) | ~10 entities |
| Reduction | **~95%** |

The J key now spawns a complete GPU explosion with only the impact flash remaining as CPU (point light + glow circle - benefits from being a single entity).

## Visual Tuning Session (December 24, 2025)

Initial GPU implementations were functional but visually "off" compared to CPU versions. The following fixes were applied:

### Smoke Cloud Fixes

**Color**: Changed from grey to brown tones matching CPU's `SmokeColorOverLife`:
```rust
// Before: Grey (wrong)
smoke_color_gradient.add_key(0.0, Vec4::new(0.4, 0.4, 0.4, 0.6));

// After: Brown interpolation (correct)
smoke_color_gradient.add_key(0.0, Vec4::new(0.147, 0.117, 0.089, 0.0));   // Dark brown
smoke_color_gradient.add_key(0.25, Vec4::new(0.192, 0.147, 0.106, 0.25));
smoke_color_gradient.add_key(0.5, Vec4::new(0.237, 0.176, 0.123, 0.5));   // Peak alpha
smoke_color_gradient.add_key(0.75, Vec4::new(0.283, 0.206, 0.139, 0.25));
smoke_color_gradient.add_key(1.0, Vec4::new(0.328, 0.235, 0.156, 0.0));   // Tan
```

**Alpha**: Changed from linear fade to smoothstep bell curve (0→0.5→0)

**Size curve**: Changed from linear to ease-out `1-(1-t)²`:
```rust
// ease_out(t) = 1 - (1-t)², scale = initial * (1 + 2*ease_out)
smoke_size_gradient.add_key(0.0, Vec3::splat(0.75));   // 1x
smoke_size_gradient.add_key(0.25, Vec3::splat(1.41));  // ease-out
smoke_size_gradient.add_key(0.5, Vec3::splat(1.88));
smoke_size_gradient.add_key(0.75, Vec3::splat(2.16));
smoke_size_gradient.add_key(1.0, Vec3::splat(2.25));   // 3x
```

### Wisp Puffs Fixes

**Alpha**: Changed from simple fade to two-phase (4→1 fast, then 1→0 linear):
```rust
wisp_color_gradient.add_key(0.0, Vec4::new(0.15, 0.12, 0.10, 4.0));   // Start HDR
wisp_color_gradient.add_key(0.2, Vec4::new(0.15, 0.12, 0.10, 1.0));   // Fast drop
wisp_color_gradient.add_key(1.0, Vec4::new(0.15, 0.12, 0.10, 0.0));   // Linear fade
```

**Size curve**: Changed from cubic ease-in (t³) to smoothstep `t²(3-2t)`:
```rust
// smoothstep(t) = t² * (3 - 2t)
wisp_size_gradient.add_key(0.0, Vec3::splat(0.0));
wisp_size_gradient.add_key(0.2, Vec3::splat(1.04));    // smoothstep(0.2) * 10
wisp_size_gradient.add_key(0.4, Vec3::splat(3.52));
wisp_size_gradient.add_key(0.6, Vec3::splat(6.48));
wisp_size_gradient.add_key(0.8, Vec3::splat(8.96));
wisp_size_gradient.add_key(1.0, Vec3::splat(10.0));
```

### Random Sprite Rotation

CPU uses `SpriteRotation { angle }` to give each billboard a random rotation (0-360°), making particles blend together rather than appearing as distinct quads.

**Key insight**: For `FaceCameraPosition`, use `.with_rotation()` NOT `.with_axis_rotation()`:
- `.with_axis_rotation()` rotates around Y axis (wrong - spins like a turntable)
- `.with_rotation()` rotates in camera plane (correct - spins like a coin facing you)

```rust
let random_rot = writer.rand(ScalarType::Float) * writer.lit(TAU);
let init_rot = SetAttributeModifier::new(Attribute::F32_0, random_rot.expr());
let rotation = writer.attr(Attribute::F32_0).expr();

.init(init_rot)
.render(OrientModifier::new(OrientMode::FaceCameraPosition)
    .with_rotation(rotation))
```

### Flipbook Animation

GPU effects were stuck on frame 0 - missing flipbook animation. Fixed by adding `SPRITE_INDEX` update each frame:

```rust
// Animate sprite index based on age/lifetime
let frame = (writer.attr(Attribute::AGE) / writer.attr(Attribute::LIFETIME) * writer.lit(64.0))
    .cast(ScalarType::Int)
    .min(writer.lit(63i32));
let update_sprite = SetAttributeModifier::new(Attribute::SPRITE_INDEX, frame.expr());

.update(update_sprite)  // Must be in update phase, not init!
```

- Smoke: 35 frames over lifetime
- Wisp: 64 frames over lifetime

### Commits

- `db5ed21` - GPU smoke/wisp: fix color, alpha, and size curves to match CPU
- `e8695d3` - GPU smoke/wisp: add random rotation and flipbook animation
