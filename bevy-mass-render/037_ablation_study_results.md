# Ablation Study: Ground Explosion FPS Bottleneck Analysis

## Problem

After migrating sparks, flash sparks, and parts debris to GPU (bevy_hanabi), scatter barrage still dropped to ~40 FPS despite ~98% entity reduction for those emitters.

## Methodology

Created `spawn_ground_explosion_gpu_only()` ablation test function that progressively adds CPU emitters to isolate performance impact.

Debug keys:
- `L`: Single ablation test explosion
- `Shift+L`: 8-explosion barrage (same as scatter artillery)

## Results

| Configuration | Entities/Explosion | FPS (8-shell barrage) |
|--------------|-------------------|----------------------|
| GPU only (sparks + flash + parts) | 3 | 90+ |
| + fireballs (main + secondary) + dust + impact | ~22-37 | 70-80 |
| + dirt debris + velocity dirt | ~42-67 | 45-50 |
| Full explosion (+ smoke + wisps) | ~60-90 | ~40 |

## Key Findings

1. **GPU particles are highly efficient**: 3 GPU entities per explosion maintain 90+ FPS even with 8 simultaneous explosions.

2. **Fireballs are acceptable**: Main (9-17) + secondary (7-13) fireballs + dust (2-3) + impact (1) only drop FPS to 70-80.

3. **Dirt emitters are major bottleneck**: Adding dirt debris (10-15) + velocity dirt (10-15) causes 25-35 FPS drop (70-80 → 45-50).

4. **Smoke adds final ~5-10 FPS cost**: Smoke (10-15) + wisps (2-3) bring it down to ~40 FPS.

## Migration Feasibility

From plan document analysis:

| Emitter | Can Migrate to GPU? | Notes |
|---------|---------------------|-------|
| Dirt debris | **Yes** | Non-uniform XY supported via Vec3 SIZE |
| Velocity dirt | **Yes** | Same as dirt debris |
| Smoke cloud | Yes | FlipbookModifier works |
| Wisps | Yes | Low complexity |

### Dirt Emitter Details

CPU implementation uses non-uniform scaling:
```rust
let base_scale_x = rng.gen_range(0.3..1.0);  // Width variation
let base_scale_y = rng.gen_range(0.4..1.0);  // Height variation
Transform::from_translation(position)
    .with_scale(Vec3::new(size * base_scale_x, size * base_scale_y, size))
```

### bevy_hanabi Non-Uniform Size Support - CONFIRMED

After exploring `/home/gota/ggando/gamedev/bevy_hanabi/`, found **full support** for non-uniform XY scaling:

1. **SIZE attribute** is stored as `Vec3` (from `attributes.rs:87-89`):
   - `Attribute::SIZE` - uniform size
   - `Attribute::SIZE2` - non-uniform 2D size
   - `Attribute::SIZE3` - non-uniform 3D size

2. **SizeOverLifetimeModifier** uses `Gradient<Vec3>` - independently interpolates X, Y, Z

3. **Shader code** (from `vfx_render.wgsl:209-210`):
   ```wgsl
   let vpos = vertex_position * size;  // Component-wise multiplication
   let sim_position = position + axis_x * vpos.x + axis_y * vpos.y + axis_z * vpos.z;
   ```

**Conclusion**: Dirt emitters CAN be fully migrated to GPU with proper non-uniform scaling. The plan document blocker "XY flatten needs shader" was incorrect.

## Recommendations

### Priority 1: Migrate Smoke Cloud
- 10-15 entities per explosion
- Uses flipbook animation (supported)
- No complex scaling requirements
- Expected gain: ~5-10 FPS

### Priority 2: Migrate Dirt Emitters
- 20-30 entities per explosion (dirt + velocity dirt)
- Major performance bottleneck
- Can simplify to uniform scale if needed
- Expected gain: ~25-30 FPS

### Priority 3: Migrate Wisps
- Only 2-3 entities, low priority
- Simple implementation

## GPU Dirt Migration (In Progress)

### Implementation

Added two GPU particle effects to replace CPU dirt entities:

1. **`ground_dirt_effect`** - Replaces `spawn_dirt_debris()` (35 entities → 1 GPU effect)
   - 35 particles, camera-facing (`OrientMode::FaceCameraPosition`)
   - Gravity 9.8 m/s² + drag 2.0
   - Non-uniform XY size via `Attribute::SIZE3`
   - Color: dark brown (0.082, 0.063, 0.050) with fade-in/out

2. **`ground_vdirt_effect`** - Replaces `spawn_velocity_dirt()` (10-15 entities → 1 GPU effect)
   - 12 particles, velocity-aligned (`OrientMode::AlongVelocity`)
   - No gravity, drag 2.0 only
   - Hemisphere cone velocity with center-weighted speed falloff
   - Non-uniform XY size with axis swap (see below)

### Axis Mapping Fix

**Critical discovery**: bevy_hanabi `AlongVelocity` interprets size differently from CPU `VelocityAligned`:
- CPU: Transform.scale.Y = along velocity (height), scale.X = perpendicular (width)
- GPU: SIZE3.X = along velocity, SIZE3.Y = perpendicular

Solution: Swap axes in GPU effect:
```rust
// CPU: base_scale_x = 0.5-1.0 (width), base_scale_y = 0.6-1.2 (height along velocity)
// GPU: Swap - GPU_X = CPU_Y, GPU_Y = CPU_X
let vdirt_size = (base_size * cpu_scale_y).vec3(base_size * cpu_scale_x, base_size);
```

### Debug Keys

| Key | CPU Emitter | GPU Version |
|-----|-------------|-------------|
| 3 | Dirt debris | Shift+3 |
| 4 | Velocity dirt | Shift+4 |

### Size Gradient Fix

**Second issue**: GPU velocity dirt was shrinking (1.0→0.3) but CPU grows (1.0→2.0).

CPU `Dirt001ScaleOverLife` (line 2236-2237):
```rust
// UE5: Linear growth from 1.0 to 2.0 (opposite of dirt!)
let growth_factor = 1.0 + t;
```

Fix: Change GPU size gradient from shrinking to growing:
```rust
// Before (wrong - shrinking)
vdirt_size_gradient.add_key(0.0, Vec3::splat(1.0));
vdirt_size_gradient.add_key(1.0, Vec3::splat(0.3));

// After (correct - growing like CPU)
vdirt_size_gradient.add_key(0.0, Vec3::splat(1.0));  // Start at 1.0×
vdirt_size_gradient.add_key(1.0, Vec3::splat(2.0)); // Grow to 2.0×
```

### Status

- ✅ GPU dirt debris (Shift+3) matches CPU (key 3) - confirmed "identical"
- ✅ GPU velocity dirt (Shift+4) matches CPU (key 4) - axis swap + size growth fixes applied

## Files Changed

| File | Changes |
|------|---------|
| `src/ground_explosion.rs` | Added `spawn_ground_explosion_gpu_only()` ablation function, L/Shift+L debug keys, Shift+3/4 for isolated GPU dirt testing |
| `src/particles.rs` | Added `ground_dirt_effect`, `ground_vdirt_effect`, `spawn_ground_explosion_gpu_dirt()` |

## Code Reference

Ablation test function at [ground_explosion.rs:533](src/ground_explosion.rs#L533):
```rust
pub fn spawn_ground_explosion_gpu_only(
    commands: &mut Commands,
    assets: &GroundExplosionAssets,
    flipbook_materials: &mut ResMut<Assets<FlipbookMaterial>>,
    additive_materials: &mut ResMut<Assets<AdditiveMaterial>>,
    gpu_effects: &ExplosionParticleEffects,
    position: Vec3,
    scale: f32,
    current_time: f64,
) {
    // GPU particles (sparks, flash sparks, parts debris)
    spawn_ground_explosion_gpu_sparks(commands, gpu_effects, position, scale, current_time);

    // GPU dirt (replaces CPU dirt_debris + velocity_dirt)
    spawn_ground_explosion_gpu_dirt(commands, gpu_effects, position, scale, current_time);

    // CPU emitters for ablation
    spawn_main_fireball(...);
    spawn_secondary_fireball(...);
    spawn_dust_ring(...);
    spawn_impact_flash(...);
}
```

GPU dirt spawn function at [particles.rs:1314](src/particles.rs#L1314):
```rust
pub fn spawn_ground_explosion_gpu_dirt(
    commands: &mut Commands,
    particle_effects: &ExplosionParticleEffects,
    position: Vec3,
    scale: f32,
    current_time: f64,
) {
    // GPU Dirt Debris (35 entities → 1 GPU effect)
    commands.spawn((
        ParticleEffect { handle: particle_effects.ground_dirt_effect.clone(), ... },
        EffectMaterial { images: vec![particle_effects.ground_dirt_texture.clone()] },
        ...
    ));

    // GPU Velocity Dirt (10-15 entities → 1 GPU effect)
    commands.spawn((
        ParticleEffect { handle: particle_effects.ground_vdirt_effect.clone(), ... },
        ...
    ));
}
```
