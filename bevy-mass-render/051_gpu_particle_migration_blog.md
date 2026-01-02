# The GPU Particle Migration: A Journey Through Pain

**A retrospective on migrating from CPU ECS entities to bevy_hanabi GPU particles**

---

## The Problem

Ground explosions in our Bevy RTS game were killing performance. Each explosion spawned **150-280 ECS entities** - individual particles for sparks, flash sparks, debris, dirt, smoke, and more. During artillery barrages with 6-10 simultaneous explosions, FPS tanked from 100 to 40.

The CPU particle system was elegant in its simplicity: each particle was a Bevy entity with a mesh, material, and transform. But at scale, the ECS overhead became crushing.

**Solution**: Migrate to bevy_hanabi GPU particles - single draw call per emitter type instead of per-particle entities.

---

## Phase 1: The Easy Wins (Sparks, Flash Sparks, Parts)

The first emitters migrated relatively smoothly, but not without learning some painful lessons.

### Lesson 1: Additive Blending is Non-Negotiable

**Symptom**: GPU sparks showed dark/brown backgrounds instead of glowing.

**Cause**: bevy_hanabi defaults to `AlphaMode::Blend`, which blends with background instead of adding light.

**Fix**:
```rust
.with_alpha_mode(bevy_hanabi::AlphaMode::Add)
```

### Lesson 2: The 4x Brightness Trap

**Symptom**: GPU particles appeared orange-red instead of hot yellow-white.

**Cause**: Our CPU shader applied a **4x brightness multiplier** that we'd forgotten about:
```wgsl
let brightness = 4.0;
let final_color = tex.rgb * tint_color.rgb * brightness;
```

CPU code pre-divided colors by 4, expecting the shader to multiply back. GPU gradients needed the final rendered values directly.

**Fix**: Apply 4x multiplier to GPU gradient values:
```rust
// Before (wrong): Vec4::new(12.5, 6.75, 1.9, 1.0)
// After (correct): Vec4::new(50.0, 27.0, 7.6, 1.0)
```

### Lesson 3: Velocity-Aligned Axis Swap

**Symptom**: Flash sparks stretched horizontally (wrong direction) in "shooting star" effect.

**Cause**: bevy_hanabi's `AlongVelocity` mode:
- **X axis** = along velocity direction (particle length)
- **Y axis** = perpendicular (particle width)

Our CPU `VelocityAligned` was opposite - Y along velocity.

**Fix**: Swap size gradient axes.

### Lesson 4: Drag ≠ Deceleration

**Symptom**: Flash sparks traveled half the CPU distance with a "weird stopping effect."

**Cause**: `LinearDragModifier` applies velocity-proportional drag (exponential decay). CPU used constant deceleration.

**Fix**: Replace drag with `AccelModifier`:
```rust
// Before (exponential slowdown)
LinearDragModifier::new(4.0)

// After (constant deceleration matching CPU)
AccelModifier::new(Vec3::new(-2.5, -10.0, -5.0))
```

### Parts Debris: Sprite Sheet Baking

For 3D debris chunks, we baked the CPU meshes into a 24-frame sprite sheet (3 variants × 8 viewing angles) using a custom Bevy tool, then used `FlipbookModifier` for random frame selection.

**Result after Phase 1**: Sparks, flash sparks, and parts debris reduced from ~100-185 entities to 3 GPU effects per explosion.

---

## Phase 2: The Ablation Study

With Phase 1 complete, barrages still dropped to ~45 FPS. Something else was eating performance.

We built an ablation test function that progressively added CPU emitters to isolate the bottleneck:

| Configuration | Entities | FPS (8-shell barrage) |
|--------------|----------|----------------------|
| GPU only (sparks + flash + parts) | 3 | 90+ |
| + fireballs + dust | ~22-37 | 70-80 |
| + dirt debris | ~42-67 | **45-50** |
| Full explosion | ~60-90 | ~40 |

**Finding**: Dirt emitters (20-30 entities) were the major bottleneck - not the high-count effects we'd already migrated.

---

## Phase 3: The Dirt Migration (Easy)

Dirt emitters used non-uniform XY scaling, which we initially thought required custom modifiers. Turns out bevy_hanabi's `SizeOverLifetimeModifier` accepts `Gradient<Vec3>` - each key can have different X, Y, Z values.

Two gotchas:
1. **Axis swap again**: GPU X = along velocity, CPU Y = along velocity
2. **Size direction reversed**: CPU velocity-dirt grew 1×→2×, GPU gradient was shrinking

**Result**: Dirt debris (35 entities) and velocity dirt (10-15 entities) reduced to 2 GPU effects.

---

## Phase 4: The Fireball Nightmare (8 Devlogs of Pain)

The fireballs broke us. What started as "should be straightforward" became 8 devlogs and eventually required **forking bevy_hanabi**.

### The Core Problem: A Visual Illusion

**Symptom**: Particles appeared to travel "inward toward center then fly out the other side."

We spent days investigating orientation math, cross product sign flips, quaternion vs axis-based rotation... but the actual particle velocities were **correct the whole time**.

**The real cause**: The 500x→1x UV zoom was so intense it completely overwhelmed the actual particle motion, creating a visual illusion of inward movement.

#### How the Illusion Works

With UV scale starting at 500x:
- Visible texture region at t=0: ~0.016m (tiny center point)
- Visible texture region at t=1: 20m (full fireball)
- **Apparent expansion rate**: ~13 m/s radially outward from texture center

Meanwhile, actual particle velocity was only 3-5 m/s. The UV zoom's apparent motion dominated, and because it expanded from center (not bottom-pivot like CPU), it created the illusion of incorrect movement.

#### The Debug Breakthrough

We confirmed velocities were correct by spawning **small colored quads** instead of large fireballs with UV zoom:
- Particles clearly moved outward
- Motion was correct all along
- The large quads + intense UV zoom had fooled our eyes

### The Actual Fixes

#### Fix 1: UVScaleOverLifetimeModifier Bug

Our custom modifier had inverted math:
```wgsl
// WRONG: multiplying sends UVs out of bounds
let uv_scaled = uv_centered * uv_scale + vec2(0.5, 0.5);
// With scale=500: UV 0.0 becomes -249.5 (samples edge/garbage)

// CORRECT: dividing zooms IN on center
let uv_scaled = uv_centered / uv_scale + vec2(0.5, 0.5);
// With scale=500: UV 0.0 becomes 0.499 (samples center)
```

This was sampling edge pixels (solid color) instead of zooming in on the center, breaking the whole effect.

#### Fix 2: UV Pivot

CPU fireballs use bottom-pivot - the UV zoom expands **upward** along velocity. Our GPU zoom expanded from center, fighting the particle motion.

Added configurable pivot to `UVScaleOverLifetimeModifier`:
```rust
UVScaleOverLifetimeModifier {
    gradient: ...,
    pivot: Vec2::new(0.5, 1.0),  // Bottom-pivot
}
```

#### Fix 3: SimulationSpace::Local

bevy_hanabi defaults to `SimulationSpace::Global`, which **ignores** `Transform.scale`. CPU velocity was `3-5 m/s * scale`, GPU was fixed.

Switched to `SimulationSpace::Local` so effect scale applies to velocities.

#### Fix 4: Spawn Hemisphere Size

Initial 0.5m radius (1m diameter) was too small - particles appeared stationary because the large 8-20m particle sizes completely obscured the outward movement. Increased to 7.5m radius.

### The Orientation Rabbit Hole (Separate Issue)

We did discover a real orientation issue along the way: bevy_hanabi's `AlongVelocity` uses X=velocity while CPU used Y=velocity. This caused texture sideways orientation (not the perceived "mirroring").

**Solution**: Use original X=velocity orientation and rotate the texture asset 90°.

The cross-product sign flip analysis in devlogs 041-044 was mathematically correct - it just wasn't causing the main visual problem we were chasing.

### Problem 3: Particles Going Inward (Expression Re-evaluation)

After fixing orientation, particles went **INWARD** instead of outward.

**Cause**: bevy_hanabi's `ExprWriter` re-evaluates `rand()` calls on every expression use. Cloning an expression doesn't preserve computed values:

```rust
let dir = rx.vec3(ry, rz).normalized();  // Uses rand()
let pos = dir.clone() * radius;          // NEW random values!
let vel = dir.clone() * speed;           // DIFFERENT random values!
// Result: pos and vel point in completely different directions
```

**Fix**: Read the POSITION attribute after it's set:
```rust
let fb_pos = rx.vec3(ry, rz).normalized() * radius;
let fireball_init_pos = SetAttributeModifier::new(Attribute::POSITION, fb_pos.expr());

let fb_pos_read = writer_fireball.attr(Attribute::POSITION);  // Read back!
let fb_velocity = fb_pos_read.normalized() * fb_speed;
```

### Problem 4: UVScaleOverLifetimeModifier Bug

**Symptom**: UV zoom made particles invisible or single-color.

**Cause**: Our custom modifier had wrong math:
```wgsl
// WRONG: multiplying by scale sends UVs out of bounds (0.0 → -249.5)
let uv_scaled = uv_centered * uv_scale + vec2(0.5, 0.5);

// CORRECT: dividing zooms IN on center
let uv_scaled = uv_centered / uv_scale + vec2(0.5, 0.5);
```

---

## Phase 5: The Cleanup (Dust Ring, Smoke, Wisps)

After the fireball trauma, the remaining emitters were refreshingly straightforward.

### Dust Ring
- Non-uniform X/Y scaling: `SizeOverLifetimeModifier` with `Gradient<Vec3>` works
- Random fixed frame: Set `SPRITE_INDEX` at init, never update

### Smoke Cloud & Wisp Puffs
Key fixes:
- **Color curves**: Grey → brown tones, matching CPU's color interpolation
- **Alpha curves**: Linear fade → smoothstep bell curve
- **Size curves**: Linear growth → ease-out `1-(1-t)²`
- **Random rotation**: For `FaceCameraPosition`, use `.with_rotation()` NOT `.with_axis_rotation()`
- **Flipbook animation**: Update `SPRITE_INDEX` each frame based on age/lifetime

---

## The Fork: bevy_hanabi Modifications

The migration required forking bevy_hanabi with these additions:

1. **Rotation support for AlongVelocity**: Apply in-plane rotation after orientation calculation
2. **axis_rotation**: Spin particle around velocity axis
3. **UVScaleOverLifetimeModifier**: Gradient-based UV scaling with configurable pivot
4. **Consistent axis_y direction**: Fix for horizontal mirroring via world-right reference

---

## Final Results

| Emitter | CPU Entities | GPU Entities | Status |
|---------|-------------|--------------|--------|
| Main Fireball | 9-17 | 1 | Done |
| Secondary Fireball | 7-13 | 1 | Done |
| Sparks | 30-60 | 1 | Done |
| Flash Sparks | 20-50 | 1 | Done |
| Parts Debris | 50-75 | 1 | Done |
| Dirt Debris | ~35 | 1 | Done |
| Velocity Dirt | 10-15 | 1 | Done |
| Dust Ring | 2-3 | 1 | Done |
| Smoke Cloud | 10-15 | 1 | Done |
| Wisp Puffs | 3 | 1 | Done |
| Impact Flash | 1 | - | CPU (single entity, not worth migrating) |

**Total reduction**: ~170-290 entities → ~10 entities per explosion (**95% reduction**)

**Performance**: 8-shell barrages now maintain 80+ FPS instead of dropping to 40.

---

## Key Takeaways

1. **Debug with simple visuals first** - Small colored quads revealed correct velocity when complex effects created illusions
2. **UV zoom can overwhelm particle motion** - 500x zoom creates ~13 m/s apparent motion that drowns out 3-5 m/s actual velocity
3. **UV zoom math is counterintuitive** - Divide to zoom in, multiply to zoom out (we had it backwards)
4. **UV pivot matters** - Center-pivot vs bottom-pivot creates completely different visual motion
5. **SimulationSpace affects scale** - Global ignores transform scale, Local applies it
6. **Additive blending matters** - Sparks need `AlphaMode::Add`, not Blend
7. **Watch for hidden multipliers** - CPU shaders often have brightness multipliers
8. **Axis conventions differ** - GPU X=velocity vs CPU Y=velocity requires texture rotation
9. **Drag ≠ deceleration** - LinearDragModifier is exponential, use AccelModifier for constant
10. **Expression cloning re-evaluates** - Use attr() to read back computed values
11. **Sometimes you need to fork** - Complex VFX may require engine modifications

---

## Timeline

- **December 16, 2025**: Started migration, completed sparks/flash/parts
- **December 17, 2025**: Ablation study, dirt migration
- **December 17-22, 2025**: The fireball nightmare (5 days, 8 devlogs)
- **December 23, 2025**: Dust ring, smoke, wisps completed
- **December 24, 2025**: Visual tuning, flipbook animation fixes

Total time: **~9 days** of intensive debugging, with ~60% spent on the fireball orientation/velocity issues alone.

Was it worth it? The 95% entity reduction and stable 80+ FPS during barrages say yes. But I'd be lying if I said I didn't consider reverting to CPU particles multiple times during the fireball saga.
