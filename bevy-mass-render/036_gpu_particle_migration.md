# GPU Particle Migration: bevy_hanabi Integration

## Overview

This devlog documents the migration of CPU-based spark particle emitters to GPU particles using bevy_hanabi for ground explosions. The goal is to reduce ECS entity overhead from ~150-280 entities per explosion.

## Problem Statement

Ground explosions cause significant FPS drops:
- Single explosion: ~100 FPS → ~80 FPS
- Scatter barrage (6-10 shells): ~100 FPS → ~40 FPS

Root cause: Each explosion spawns ~150-280 ECS entities, each with its own mesh, material, and transform. Heavy emitters include:
- Sparks: 30-60 particles (CPU entities)
- Flash sparks (spark_l): 20-50 particles (CPU entities)
- Parts debris: 50-75 particles (CPU entities)

## Solution: bevy_hanabi GPU Particles

Migrate high-count emitters from CPU ECS entities to bevy_hanabi GPU particles:
- Single draw call per emitter type instead of per-particle
- GPU-based simulation instead of CPU systems
- No per-entity Transform/Material overhead

## Implementation Progress

### Phase 1: GPU Sparks (`ground_sparks_effect`)

**CPU Implementation** (`SparkColorOverLife` in ground_explosion.rs):
- Count: 30-60 particles
- Spawn: 90° upward cone from explosion center
- Velocity: 15-37.5 m/s radially outward
- Lifetime: 0.5-2.0s
- Physics: Gravity at 9.8 m/s²
- Color: HDR (50, 27, 7.6) → (1, 0.05, 0) over 55% lifetime, then fade
- Texture: flare.png
- Orientation: VelocityAligned

**GPU Implementation** (particles.rs):
```rust
// Cone spawn position (90° cone)
let spark_init_pos = SetPositionCone3dModifier {
    height: writer_spark.lit(1.0).expr(),
    base_radius: writer_spark.lit(1.0).expr(),  // 45° half-angle = 90° cone
    top_radius: writer_spark.lit(0.0).expr(),
    dimension: ShapeDimension::Surface,
};

// Radial velocity from spawn position
let spark_init_vel = SetVelocitySphereModifier {
    center: writer_spark.lit(Vec3::ZERO).expr(),
    speed: (writer_spark.lit(15.0) + writer_spark.rand(ScalarType::Float) * writer_spark.lit(22.5)).expr(),
};

// HDR color gradient for bloom
let mut spark_color_gradient = bevy_hanabi::Gradient::new();
spark_color_gradient.add_key(0.0, Vec4::new(12.5, 6.75, 1.9, 1.0));
spark_color_gradient.add_key(0.55, Vec4::new(0.25, 0.0125, 0.0, 0.5));
spark_color_gradient.add_key(1.0, Vec4::new(0.125, 0.006, 0.0, 0.0));

// Effect with additive blending
EffectAsset::new(512, SpawnerSettings::once(45.0.into()), spark_module)
    .with_name("ground_explosion_sparks")
    .with_alpha_mode(bevy_hanabi::AlphaMode::Add)  // Critical for spark glow
    // ...modifiers...
```

### Phase 2: GPU Flash Sparks (`ground_flash_sparks_effect`)

**CPU Implementation** (`SparkLColorOverLife` in ground_explosion.rs):
- Count: 20-50 particles
- Spawn: Ring pattern at equator (100° cone)
- Velocity: 4-55 m/s radially outward
- Lifetime: 0.3-1.0s
- Physics: Deceleration (-0.25, -1.0, -0.5) instead of gravity
- Color: Constant HDR (10, 6.5, 3.9), alpha fades
- Scale: "Shooting star" effect - elongated at spawn, normalizes over time
- Orientation: VelocityAligned

**GPU Implementation** (particles.rs):
```rust
// 100° cone (slightly wider than sparks)
let flash_init_pos = SetPositionCone3dModifier {
    height: writer_flash.lit(1.0).expr(),
    base_radius: writer_flash.lit(1.2).expr(),  // ~50° half-angle
    top_radius: writer_flash.lit(0.0).expr(),
    dimension: ShapeDimension::Surface,
};

// Shooting star size gradient
// For AlongVelocity: X = along velocity (length), Y = perpendicular (width)
let mut flash_size_gradient = bevy_hanabi::Gradient::new();
flash_size_gradient.add_key(0.0, Vec3::new(10.0, 0.06, 1.0));   // Elongated along velocity
flash_size_gradient.add_key(0.05, Vec3::new(10.0, 0.06, 1.0));  // Hold briefly
flash_size_gradient.add_key(0.5, Vec3::new(0.6, 1.0, 1.0));     // Normalize
flash_size_gradient.add_key(1.0, Vec3::new(0.6, 1.0, 1.0));

// Drag for deceleration effect
let flash_update_drag = LinearDragModifier::new(writer_flash.lit(3.0).expr());
```

## Debug Menu Changes

### Key Remapping

| Key | Before | After |
|-----|--------|-------|
| `0` | WFX toggle | Parts (CPU) |
| `O` | - | WFX toggle |
| `8` | - | Spark (CPU) |
| `9` | - | Spark_l (CPU) |
| `Shift+8` | - | Spark (GPU) |
| `Shift+9` | - | Spark_l (GPU) |

### Usage
1. Press `P` to enter ground explosion debug mode
2. Press `8` for CPU spark, `Shift+8` for GPU spark
3. A/B compare visually at same position

## Technical Learnings

### 1. Additive Blending is Critical

**Issue**: GPU particles showed dark/brown background instead of glowing sparks.

**Fix**: Must use `AlphaMode::Add` for spark effects:
```rust
.with_alpha_mode(bevy_hanabi::AlphaMode::Add)
```

Default `AlphaMode::Blend` causes sparks to blend with background instead of adding light.

### 2. Velocity-Aligned Size Axis Orientation

**Issue**: Spark_l particles stretched horizontally (wrong direction) when using `OrientMode::AlongVelocity`.

**Understanding**: In bevy_hanabi's `AlongVelocity` mode:
- **X axis** = along velocity direction (particle length/stretch)
- **Y axis** = perpendicular to velocity (particle width)

**Fix**: Size gradient changed from `(width, length, z)` to `(length, width, z)`:
```rust
// Wrong: (0.06, 10.0, 1.0) - thin width, long "height"
// Correct: (10.0, 0.06, 1.0) - long along velocity, thin perpendicular
flash_size_gradient.add_key(0.0, Vec3::new(10.0, 0.06, 1.0));
```

### 3. Cone Spawn vs Sphere Spawn

**Issue**: GPU sparks had full spherical spread instead of upward cone.

**Fix**: Use `SetPositionCone3dModifier` instead of `SetPositionSphereModifier`:
```rust
let spark_init_pos = SetPositionCone3dModifier {
    height: writer_spark.lit(1.0).expr(),
    base_radius: writer_spark.lit(1.0).expr(),
    top_radius: writer_spark.lit(0.0).expr(),
    dimension: ShapeDimension::Surface,
};
```

### 4. Texture Binding via EffectMaterial

bevy_hanabi requires texture binding through `EffectMaterial` component:
```rust
commands.spawn((
    ParticleEffect::new(effect_handle),
    EffectMaterial {
        images: vec![texture_handle.clone()],
    },
    // ...
));
```

The texture slot index in `ParticleTextureModifier` corresponds to the index in `EffectMaterial.images`.

## Files Modified

| File | Changes |
|------|---------|
| `src/particles.rs` | Added `ground_sparks_effect`, `ground_flash_sparks_effect`, `ground_sparks_texture` to `ExplosionParticleEffects`; GPU effect definitions |
| `src/ground_explosion.rs` | Debug menu key remapping; Shift+8/9 for GPU spawns |
| `src/objective.rs` | WFX toggle moved from `0` to `O` key |

## Session 2: Visual Parity Fixes (2025-12-16)

### Problem: GPU vs CPU Color Mismatch

GPU particles appeared orange-red instead of the hot yellow-white of CPU sparks.

**Root Cause**: The CPU shader (`wfx_additive.wgsl`) applies a **4x brightness multiplier**:
```wgsl
let brightness = 4.0;
let final_color = vec4<f32>(
    tex.rgb * tint_color.rgb * brightness * soft_alpha * tint_color.a,
    final_alpha
);
```

The CPU code pre-divides color values by 4, expecting the shader to multiply back:
- CPU spark color: `(12.5, 6.75, 1.9)` (pre-divided)
- Final rendered: `(50, 27, 7.6)` (after 4x shader multiplier)

**Fix**: Apply 4x multiplier directly to GPU gradient values:
```rust
// Before (wrong - too dim)
spark_color_gradient.add_key(0.0, Vec4::new(12.5, 6.75, 1.9, 1.0));

// After (correct - matches CPU rendered output)
spark_color_gradient.add_key(0.0, Vec4::new(50.0, 27.0, 7.6, 1.0));
```

Same fix for flash sparks:
```rust
// CPU: (2.5, 1.625, 0.975) * 4 = (10, 6.5, 3.9)
flash_color_gradient.add_key(0.0, Vec4::new(10.0, 6.5, 3.9, 1.0));
```

### Problem: Flash Sparks Wrong Orientation

Flash sparks appeared horizontal or vertical instead of velocity-aligned "shooting star" streaks.

**Root Cause**: bevy_hanabi's `OrientMode::AlongVelocity` uses:
- **X-axis = along velocity** (where the particle stretches)
- **Y-axis = perpendicular** (particle width)

CPU's `VelocityAligned` uses Y-axis along velocity. The size gradient axes were swapped.

**Fix**: Correct size gradient for `AlongVelocity`:
```rust
// Before (wrong - Y elongated, but Y is perpendicular in AlongVelocity)
flash_size_gradient.add_key(0.0, Vec3::new(0.06, 10.0, 1.0));

// After (correct - X elongated along velocity direction)
flash_size_gradient.add_key(0.0, Vec3::new(10.0, 0.1, 1.0));
```

### Summary of Changes

| Effect | Fix | Details |
|--------|-----|---------|
| Spark (8) | 4x color multiplier | `(12.5, 6.75, 1.9)` → `(50, 27, 7.6)` |
| Flash Spark (9) | 4x color multiplier | `(2.5, 1.625, 0.975)` → `(10, 6.5, 3.9)` |
| Flash Spark (9) | Size axis swap | X = along velocity (elongated), Y = perpendicular (thin) |

### Texture Sampling Mode

Both effects use `ImageSampleMapping::ModulateOpacityFromR`:
- Texture R channel controls alpha/opacity only
- Color comes purely from the gradient
- Matches CPU shader's luminance-based alpha calculation

### Problem: Flash Sparks Travel Distance and Stopping Effect

GPU flash sparks traveled half the distance of CPU and had a "weird stopping effect" near the end.

**Root Cause**: Used `LinearDragModifier` which applies velocity-proportional drag:
```
velocity *= (1.0 - drag * dt)  // exponential decay
```

But CPU uses **constant deceleration** in `update_spark_l_physics`:
```rust
let deceleration = Vec3::new(-0.25, -1.0, -0.5);
velocity_aligned.velocity += deceleration * dt * 10.0;  // = (-2.5, -10, -5) m/s²
```

**Fix**: Replace `LinearDragModifier` with `AccelModifier`:
```rust
// Before (wrong - exponential slowdown)
let flash_update_drag = LinearDragModifier::new(writer_flash.lit(4.0).expr());

// After (correct - constant deceleration matching CPU)
let flash_update_accel = AccelModifier::new(
    writer_flash.lit(Vec3::new(-2.5, -10.0, -5.0)).expr()
);
```

Also fixed alpha curve to pure linear fade:
```rust
// Before (non-linear)
flash_color_gradient.add_key(0.0, Vec4::new(10.0, 6.5, 3.9, 1.0));
flash_color_gradient.add_key(0.3, Vec4::new(10.0, 6.5, 3.9, 0.7));
flash_color_gradient.add_key(0.6, Vec4::new(10.0, 6.5, 3.9, 0.4));

// After (linear 1.0 - t)
flash_color_gradient.add_key(0.0, Vec4::new(10.0, 6.5, 3.9, 1.0));
flash_color_gradient.add_key(0.5, Vec4::new(10.0, 6.5, 3.9, 0.5));
flash_color_gradient.add_key(1.0, Vec4::new(10.0, 6.5, 3.9, 0.0));
```

## Session 3: GPU Parts Debris with Sprite Sheet (2025-12-17)

### Problem: Flat Billboards Look Bad

Initial GPU parts implementation used flat colored billboards, but they looked significantly worse than CPU 3D mesh debris.

**Solution**: Bake 3D debris meshes into a sprite sheet from multiple viewing angles, then use flipbook animation to select random frames.

### Sprite Sheet Generation Tool

Created `src/bin/render_debris_sprites.rs` - a Bevy utility that renders 3D meshes to PNG:

```rust
// 3 mesh variants (same as CPU implementation)
let debris_meshes = [
    meshes.add(Cuboid::new(1.0, 0.8, 0.6)),  // Chunky rock
    meshes.add(Cuboid::new(1.2, 0.4, 0.8)),  // Flat slab
    meshes.add(Cuboid::new(0.5, 0.5, 1.4)),  // Elongated shrapnel
];

// Camera with transparent background
commands.spawn((
    Camera3d::default(),
    Camera {
        clear_color: ClearColorConfig::Custom(Color::NONE),
        ..default()
    },
    Transform::from_xyz(0.0, 0.8, 2.5).looking_at(Vec3::ZERO, Vec3::Y),
));
```

State machine captures 3 variants × 8 angles = 24 frames with proper timing delays.

**Usage**:
```bash
cargo run --bin render_debris_sprites
# Outputs: assets/textures/generated/debris_v{0-2}_a{0-7}.png

# Combine into sprite sheet with ImageMagick
cd assets/textures/generated
montage debris_v0_a*.png debris_v1_a*.png debris_v2_a*.png \
    -tile 8x3 -geometry +0+0 debris_sprites_raw.png

# Add transparency (Bevy screenshots don't preserve alpha)
convert debris_sprites_raw.png -fuzz 10% -transparent black debris_sprites.png
```

### GPU Parts Effect with Flipbook

```rust
// Load sprite sheet
let ground_parts_texture: Handle<Image> =
    asset_server.load("textures/generated/debris_sprites.png");

// Random sprite index [0, 23] at spawn
let parts_init_sprite = SetAttributeModifier::new(
    Attribute::SPRITE_INDEX,
    (writer_parts.rand(ScalarType::Float) * writer_parts.lit(24.0))
        .cast(ScalarType::Int)
        .expr()
);

// Add texture slot
let mut parts_module = writer_parts.finish();
parts_module.add_texture_slot("debris_sprites");

// Effect with flipbook
EffectAsset::new(128, SpawnerSettings::once(60.0.into()), parts_module)
    .with_name("ground_explosion_parts")
    .with_alpha_mode(bevy_hanabi::AlphaMode::Blend)
    .init(parts_init_sprite)
    // ... other modifiers
    .render(ParticleTextureModifier {
        texture_slot: parts_texture_slot,
        sample_mapping: ImageSampleMapping::Modulate,
    })
    .render(FlipbookModifier { sprite_grid_size: UVec2::new(8, 3) })
```

**Spawn with EffectMaterial**:
```rust
commands.spawn((
    ParticleEffect {
        handle: particle_effects.ground_parts_effect.clone(),
        prng_seed: Some(seed),
    },
    EffectMaterial {
        images: vec![particle_effects.ground_parts_texture.clone()],
    },
    Transform::from_translation(position).with_scale(Vec3::splat(scale)),
));
```

### Technical Notes

1. **Screenshot Timing**: Bevy needs several frames to initialize the render pipeline. Use state machine with frame delays:
   - 10 frames at startup
   - 3 frames between setup and capture

2. **Alpha Channel**: Bevy screenshots don't preserve alpha. Post-process with ImageMagick `-transparent black`.

3. **Sprite Index**: Must use `Attribute::SPRITE_INDEX` (not custom attribute) for `FlipbookModifier` to work.

4. **Grid Layout**: `sprite_grid_size: UVec2::new(8, 3)` = 8 columns × 3 rows = 24 frames.

### Debug Keys

| Key | Effect |
|-----|--------|
| `0` | Parts (CPU) |
| `Shift+0` | Parts (GPU) |

### Files Changed

| File | Changes |
|------|---------|
| `src/bin/render_debris_sprites.rs` | NEW: Sprite sheet generation tool |
| `src/particles.rs` | GPU parts effect with flipbook, `ground_parts_texture` handle |
| `src/ground_explosion.rs` | Shift+0 for GPU parts debug spawn |
| `assets/textures/generated/debris_sprites.png` | Generated 8×3 sprite sheet |

### Result

GPU parts debris is visually indistinguishable from CPU version while reducing 50-75 ECS entities to a single GPU effect.

## Remaining Issues

Fine-tuning may still be needed for:
- Exact spawn distribution (hemisphere vs cone)
- Velocity magnitude matching
- Size scale factors

## Plan Document Reference

Full migration plan with all phases documented at:
`/home/gota/.claude/plans/witty-scribbling-hopcroft.md`

## Summary: Entity Reduction

| Emitter | CPU Entities | GPU Entities | Reduction |
|---------|--------------|--------------|-----------|
| Sparks | 30-60 | 1 | ~98% |
| Flash Sparks | 20-50 | 1 | ~98% |
| Parts Debris | 50-75 | 1 | ~98% |
| **Total per explosion** | **100-185** | **3** | **~98%** |

## Next Steps

1. Performance validation with scatter barrage
2. Integrate GPU effects into main explosion spawn (replace CPU emitters)
3. Remove deprecated CPU spark/parts systems after validation
