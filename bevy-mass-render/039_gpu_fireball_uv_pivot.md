# GPU Fireball UV Zoom Pivot Investigation

## Problem

GPU fireballs look slow despite having higher velocity (6-10 m/s) than CPU fireballs (3-5 m/s). User reported scatter looks "garbage" and too slow.

## Root Cause Analysis

### Issue 1: UV Zoom Creates Competing Visual Motion

GPU fireballs have aggressive UV zoom 500x → 1x over lifetime. This creates an **apparent radial expansion of ~13 m/s** from the center:
- Visible texture region at t=0: 8m / 500 = 0.016m (tiny center point)
- Visible texture region at t=1: 20m / 1 = 20m (full fireball)
- Apparent expansion: ~20m / 1.5s = ~13 m/s

When UV zooms from center uniformly, this expansion competes with translational movement, making the actual scatter look slower.

CPU fireballs have **UV zoom disabled** (`uv_scale: 1.0`), so the scatter movement is clearly visible.

### Fix 1: UV Pivot

Modified `UVScaleOverLifetimeModifier` in bevy_hanabi fork to support configurable pivot point:

```rust
pub struct UVScaleOverLifetimeModifier {
    pub gradient: Gradient<Vec2>,
    /// Pivot point for UV scaling (0-1). Default (0.5, 0.5) = center.
    /// Use (0.5, 1.0) for bottom-pivot (expansion appears to go upward).
    pub pivot: Vec2,
}
```

Set GPU fireball to use bottom pivot `(0.5, 1.0)` so UV zoom expands along velocity direction.

**Result**: Partial improvement, but still looks slow.

### Issue 2: SimulationSpace::Global Ignores Transform Scale

**Root cause found**: bevy_hanabi defaults to `SimulationSpace::Global`, which means:
- Particle positions are in world space
- Particle velocities are in world space units
- **Transform.scale on the ParticleEffect entity is IGNORED**

CPU velocity: `3-5 m/s * scale` (ground_explosion.rs:832) - scale factor applied
GPU velocity: `3-5 m/s` (fixed) - scale factor NOT applied because Global space ignores transform

When spawning with `Transform::from_translation(position).with_scale(Vec3::splat(scale))`, the scale has no effect on Global space simulation!

### Fix 2: SimulationSpace::Local

Changed GPU fireball effect to use `SimulationSpace::Local`:

```rust
EffectAsset::new(64, SpawnerSettings::once(25.0.into()), fireball_module)
    .with_simulation_space(SimulationSpace::Local)  // Transform scale now applies!
    // ...
```

With Local space:
- Particle positions are relative to effect transform
- Transform.scale is applied to positions AND velocities
- Effect at scale 2.0 will have particles moving 2x faster in world space

## Files Changed

### bevy_hanabi fork (branch: custom-mods)

**src/modifier/output.rs**:
- Added `pivot: Vec2` field to `UVScaleOverLifetimeModifier`
- Default pivot is `(0.5, 0.5)` (center)
- Shader code now uses pivot instead of hardcoded center
- Added manual `Hash` impl to include pivot in hash

### bevy-mass-render

**src/particles.rs**:
- Set GPU fireball UV pivot to `(0.5, 1.0)` (bottom)
- Added `.with_simulation_space(SimulationSpace::Local)` to fireball effect
- Velocity at 3-5 m/s (now properly scaled by Transform.scale)

## Technical Notes

### bevy_hanabi SimulationSpace

- `SimulationSpace::Global` (default): Particles in world space, transform ignored
- `SimulationSpace::Local`: Particles relative to effect transform, scale/rotation applied

The shader handles this via `LOCAL_SPACE_SIMULATION` define:
```wgsl
fn transform_position_simulation_to_world(sim_position: vec3<f32>) -> vec4<f32> {
#ifdef LOCAL_SPACE_SIMULATION
    let transform = unpack_compressed_transform(spawner.transform);
    return transform * vec4<f32>(sim_position, 1.0);  // Scale applied!
#else
    return vec4<f32>(sim_position, 1.0);  // No transform
#endif
}
```

### Issue 3: Bottom Pivot Size Growth Creates Apparent Velocity

**Why 3-5 m/s looks fast on CPU but slow on GPU:**

CPU uses `BottomPivot` component with `bottom_pivot_quad` mesh. When the fireball scales up via `FireballScaleOverLife`, it grows **upward from its base**, not uniformly from center.

CPU fireball math:
- Velocity: 3-5 m/s (avg 4 m/s) upward
- Size growth: 7-9m → 18-23m = ~11m growth over 1.5s = 7.3 m/s growth rate
- With bottom pivot, top edge moves at: velocity + size_growth = **4 + 7.3 = 11.3 m/s apparent**

GPU fireball uses bevy_hanabi's `SizeOverLifetimeModifier` which scales from **center**, not bottom:
- Size growth: 8m → 20m = 12m growth over 1.5s = 8 m/s growth rate
- Top edge: velocity + (growth/2) = 4 + 4 = 8 m/s
- Bottom edge: velocity - (growth/2) = 4 - 4 = **0 m/s** (barely moves!)

The GPU bottom edge appearing stationary makes the whole effect look "stuck" or slow.

### Fix 3: Increase GPU Velocity to Compensate

Since GPU lacks bottom-pivot size scaling, increase velocity to match CPU's apparent top-edge motion:

```rust
// CPU: 4 m/s velocity + 7 m/s (size growth from bottom pivot) = 11 m/s apparent
// GPU: need ~11 m/s base velocity since size growth is centered
let fb_speed = writer_fireball.lit(10.0) + writer_fireball.rand(ScalarType::Float) * writer_fireball.lit(4.0); // 10-14 m/s
```

## Status

- UV pivot fix: Implemented
- SimulationSpace::Local fix: Implemented
- Velocity compensation (10-14 m/s): Implemented
- Needs visual testing to confirm fixes work together
