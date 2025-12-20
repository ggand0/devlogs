# GPU Dirt Emitter Migration

## Summary

Migrated dirt debris emitters from CPU entities to GPU particles (bevy_hanabi), completing the major performance bottleneck identified in the ablation study.

## Changes

### GPU Effects Added

1. **`ground_dirt_effect`** - Replaces `spawn_dirt_debris()` (35 entities → 1 GPU effect)
   - 35 particles, camera-facing
   - Gravity 9.8 m/s² + drag 2.0
   - Non-uniform XY size via `Attribute::SIZE3`
   - Dark brown color with fade-in/out

2. **`ground_vdirt_effect`** - Replaces `spawn_velocity_dirt()` (10-15 entities → 1 GPU effect)
   - 12 particles, velocity-aligned (`OrientMode::AlongVelocity`)
   - No gravity, drag 2.0 only
   - Hemisphere cone velocity with center-weighted speed falloff

### Fixes Required

**Axis mapping**: bevy_hanabi `AlongVelocity` uses X for along-velocity, but CPU `VelocityAligned` uses Y. Fixed by swapping X/Y scale factors.

**Size gradient**: CPU velocity dirt grows 1×→2× over lifetime, but GPU was shrinking. Fixed gradient direction.

## Performance Results

| Metric | Before (CPU dirt) | After (GPU dirt) |
|--------|-------------------|------------------|
| Entities/explosion | ~145-235 | 5 |
| FPS (7-shell barrage) | ~40 | ~60-70 |
| Improvement | - | +20-30 FPS |

## Entity Reduction Summary

| Emitter | CPU | GPU | Reduction |
|---------|-----|-----|-----------|
| Sparks | 30-60 | 1 | ~98% |
| Flash sparks | 20-50 | 1 | ~98% |
| Parts debris | 50-75 | 1 | ~98% |
| Dirt debris | 35 | 1 | ~97% |
| Velocity dirt | 10-15 | 1 | ~92% |
| **Total** | **145-235** | **5** | **~98%** |

## What's Next

Remaining CPU emitters that could be migrated:

| Emitter | Entities | Priority | Notes |
|---------|----------|----------|-------|
| Smoke cloud | 10-15 | Medium | Uses 8x8 flipbook animation |
| Wisps | 2-3 | Low | Simple, few entities |
| Main fireball | 9-17 | Low | Complex UV zoom animation |
| Secondary fireball | 7-13 | Low | Complex UV zoom animation |
| Dust ring | 2-3 | Low | Few entities |
| Impact flash | 1 | Skip | Single entity |

**Recommended next step**: Smoke cloud migration - moderate entity count, uses standard flipbook which bevy_hanabi supports via `FlipbookModifier`.

## Files Changed

- `src/particles.rs` - Added GPU dirt effects and spawn function
- `src/ground_explosion.rs` - Integrated GPU dirt into main explosion spawn

---

## GPU Fireball Migration (WIP)

### Problem: Orientation 90° Off

bevy_hanabi's `OrientMode::AlongVelocity` orients the X axis along velocity, but the CPU `VelocityAligned` mode orients Y along velocity. The fireball texture has "up" along its Y axis, so GPU fireballs appeared rotated 90° sideways.

### Solution: Fork bevy_hanabi

Created fork at `~/ggando/gamedev/bevy_hanabi` on branch `custom-mods` with two modifications:

1. **Rotation support for AlongVelocity** (commit 23f5814)
   - Added `if let Some(rotation)` check to `AlongVelocity` branch in `apply_render()`
   - Applies sin/cos rotation to swap X/Y axes when rotation is specified
   - Using `writer.lit(FRAC_PI_2).expr()` (90°) rotates Y to align along velocity

2. **PivotModifier** (commit 2a00d69)
   - New modifier for bottom-pivot particle positioning
   - Offsets position by `axis_y * size.y * 0.5` so particles grow upward from base
   - Currently disabled due to visual issues - needs debugging

### Current GPU Fireball Status

- **Working**: Orientation correct with 90° rotation, particles scatter in hemisphere
- **Working**: Per-particle random spin around velocity axis (matching CPU `SpriteRotation`)
- **Working**: UV zoom 500x→1x over lifetime (matching CPU `uv_scale` animation)
- **Issue**: Visual still not perfect - may need further tuning of size/velocity/timing
- **Disabled**: PivotModifier causes spiral/clustered appearance

### bevy_hanabi Fork Changes (branch: custom-mods)

**Commit 23f5814**: Rotation support for AlongVelocity
- Added `if let Some(rotation)` to apply in-plane rotation
- 90° rotation makes Y axis point along velocity (matching CPU VelocityAligned)

**Commit 2a00d69**: PivotModifier (currently disabled)
- Offsets position by `axis_y * size.y * 0.5` for bottom-pivot
- Causes visual issues - needs investigation

**Commit 4b91d82**: axis_rotation and UVScaleOverLifetimeModifier
- `axis_rotation`: Spins quad around velocity axis (axis_y after 90° rotation)
  - Rotates axis_x and axis_z around axis_y
  - Matches CPU `Quat::from_rotation_y(sprite_rot.angle)`
- `UVScaleOverLifetimeModifier`: Gradient-based UV scaling over lifetime
  - Scales UVs around center (0.5, 0.5) of flipbook cell
  - Enables zoom-out effect (500x→1x) matching CPU fireball behavior

### GPU Fireball Implementation

```rust
// particles.rs - GPU fireball effect
OrientModifier::new(OrientMode::AlongVelocity)
    .with_rotation(FRAC_PI_2)           // Y along velocity
    .with_axis_rotation(random_spin)     // Per-particle spin

UVScaleOverLifetimeModifier {
    gradient: 500.0 → 1.0 (smoothstep)   // UV zoom out
}

SizeOverLifetimeModifier {
    gradient: 8m → 20m                   // Physical size growth
}
```

### Remaining Issues

1. PivotModifier causes visual artifacts - disabled for now
2. Overall appearance still slightly different from CPU version
3. May need to tune velocity (currently 6-10 m/s) or size curve
