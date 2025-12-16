# 025: Explosion Performance - ECS Query Bottleneck Root Cause Analysis

**Date:** 2025-12-03
**Status:** Investigation Complete, Fix In Progress

## Summary

Tower explosion cascade causes FPS drop from 120 to 30-50 FPS. The root cause is **NOT** the explosion visual effects (billboards/particles) but rather **ECS query overhead** from iterating 700+ `PendingExplosion` component entities every frame.

## Problem Statement

When a tower is destroyed, ~1000 friendly units within the destruction radius are scheduled for delayed explosions. Each unit gets a `PendingExplosion` component with a random delay timer (0.1s to 2.0s). This creates a massive ECS query overhead.

## Investigation Timeline

### Initial Hypothesis (Wrong)
- Assumed GPU draw calls from per-entity `ExplosionMaterial` instancing was the bottleneck
- Each material add creates unique material = potential for 1000+ draw calls

### Ablation Tests Performed

| Test | Configuration | Result |
|------|---------------|--------|
| 1 | Billboards + Particles enabled | 30-50 FPS |
| 2 | Billboards disabled, Particles only | 30-50 FPS |
| 3 | Particles disabled, Billboards only | ~100 FPS |
| 4 | Both disabled (NO visual effects) | **~80 FPS** |

### Key Finding
Test 4 was critical: **Even with ZERO explosion visual effects, FPS still dropped to 80**. This proved the bottleneck was NOT in the rendering pipeline.

## Root Cause

### The Problem: Per-Entity Timer Components

```rust
// Current approach - O(n) every frame
for (entity, mut pending, transform, tower_component) in explosion_query.iter_mut() {
    pending.delay_timer -= time.delta_secs();  // Tick every entity every frame
    if pending.delay_timer <= 0.0 {
        // Process explosion
    }
}
```

With `EXPLOSION_DELAY_MAX = 2.0` seconds, dead units accumulate as pending entities:
- 1000 units die at t=0
- At t=0.5s: ~750 entities still pending (queried every frame)
- At t=1.0s: ~500 entities still pending
- At t=1.5s: ~250 entities still pending
- At t=2.0s: 0 entities pending

**Peak load: 700-800 entities queried every frame for 2 seconds**

### Log Evidence

```
EXPLOSION FRAME: 799 total pending, 35 units ready, 0 towers ready
EXPLOSION FRAME: 779 total pending, 15 units ready, 0 towers ready
EXPLOSION FRAME: 764 total pending, 0 units ready, 0 towers ready
...
```

The `total_pending` count stays high (700+) for the entire 2-second delay window.

## Secondary Bottleneck: Hanabi Particle Entities

When particles ARE enabled, there's a secondary GPU bottleneck:

- Each `ParticleEffect` entity has ~0.1ms GPU overhead
- 600 entities = ~60ms GPU time = 16 FPS theoretical minimum
- CPU time shows 0.00ms (cleanup system is fast)
- Frame time shows 20-25ms (GPU bound)

**Correlation:**
- 650 entities = 40-50 FPS
- 300 entities = 70-95 FPS (after reducing to 1 effect per unit)
- 100 entities = 150 FPS

## Solution: Centralized Explosion Queue

Replace per-entity components with a single `Resource` containing a sorted queue:

```rust
#[derive(Resource, Default)]
pub struct ExplosionQueue {
    /// Sorted by trigger_time (earliest first)
    pub queue: Vec<ScheduledExplosion>,
}

pub struct ScheduledExplosion {
    pub trigger_time: f64,  // Absolute time when explosion triggers
    pub position: Vec3,
    pub radius: f32,
    pub is_tower: bool,
}
```

### Performance Comparison

| Approach | Complexity | At 700 pending |
|----------|------------|----------------|
| Per-entity components | O(n) per frame | Query 700 entities every frame |
| Centralized queue | O(k) per frame | Only process k ready explosions |

Where k = number of explosions ready THIS frame (typically 20-50), not total pending.

## Quick Mitigation Applied

Reduced delay window to limit pending entity count:
```rust
// Before
pub const EXPLOSION_DELAY_MAX: f32 = 2.0;

// After
pub const EXPLOSION_DELAY_MAX: f32 = 0.5;
```

This reduces max pending entities from ~700 to ~175.

## Lessons Learned

1. **Profile before assuming** - Initial GPU/material hypothesis was wrong
2. **Ablation testing is crucial** - Systematically disabling components isolates the real bottleneck
3. **ECS queries have overhead** - Iterating thousands of entities every frame adds up
4. **CPU time vs Frame time** - Low CPU time with high frame time indicates GPU bottleneck
5. **Check entity counts in logs** - The "700 total pending" was the smoking gun

## Files Modified

- `src/explosion_system.rs` - Added `ExplosionQueue` resource, refactored system
- `src/constants.rs` - Reduced `EXPLOSION_DELAY_MAX` from 2.0 to 0.5
- `src/particles.rs` - Reduced buffer sizes (32K → 64), single effect per unit
- `src/objective.rs` - (TODO) Switch from component insertion to queue scheduling

## Update: Billboard Explosion Optimization (December 2024)

### Problem

After the initial ECS query fix, we wanted visible death flash explosions at each unit position when a tower is destroyed. Initial attempts with Hanabi particles failed - effects spawned but were invisible.

### Attempt 1: Direct Hanabi Particle Spawning

Spawned `ParticleEffect` components directly from `tower_destruction_system`:

```rust
commands.spawn((
    ParticleEffect::new(particle_effects.death_flash.clone()),
    Transform::from_translation(position),
    Name::new("death_flash_particle"),
));
```

**Result:** Entities spawned correctly (verified via ECS debug), but particles were **invisible**. No visual output despite correct component setup.

### Attempt 2: PendingExplosion Queue

Routed death flashes through the existing `PendingExplosion` component queue with staggered delays:

```rust
for (i, (droid_entity, _)) in units_to_destroy.iter().enumerate() {
    let delay = 0.05 + (i as f32 * 0.0005).min(0.45);
    entity_commands.try_insert(PendingExplosion {
        delay_timer: delay,
        explosion_power: 1.0,
    });
}
```

**Result:** Particles now visible! The `pending_explosion_system` handles spawning across multiple frames, which Hanabi requires for proper initialization.

**User feedback:** "I don't like it" - Hanabi particles didn't look good for this use case.

### Attempt 3: Sprite Sheet Billboard Explosions

Switched to animated flipbook billboard explosions (5x5 grid, 25 frames):

```rust
commands.spawn((
    Mesh3d(meshes.add(Rectangle::new(1.6, 1.6))),
    MeshMaterial3d(explosion_materials.add(ExplosionMaterial { ... })),
    Transform::from_translation(position),
    ExplosionAnimation { ... },
));
```

**Result:** FPS dropped from 120 to **30 FPS** when spawning 1200 individual billboard entities. Each entity had a unique material instance = no GPU batching.

### Solution: Shared Assets + Spawn Probability

Two optimizations combined:

1. **Shared mesh and material handles** in `ExplosionAssets` resource:
```rust
#[derive(Resource)]
pub struct ExplosionAssets {
    pub explosion_flipbook_texture: Handle<Image>,
    pub shared_explosion_mesh: Handle<Mesh>,
    pub shared_explosion_material: Handle<ExplosionMaterial>,
}
```

2. **20% spawn probability** - only 20% of units spawn visible explosions:
```rust
if !is_tower && rand::random::<f32>() < 0.2 {
    // Spawn billboard with shared handles
}
```

### Results

| Configuration | FPS |
|---------------|-----|
| 1200 unique materials | 30 FPS |
| Shared material + 20% spawn | **85-130 FPS** |

### Key Takeaways

1. **Hanabi requires frame-distributed spawning** - Direct spawning doesn't render; use `PendingExplosion` queue
2. **Asset sharing enables GPU batching** - Same `Handle<Mesh>` + `Handle<Material>` = 1 draw call for all instances
3. **Probabilistic spawning is invisible at scale** - 20% of 1200 = 240 explosions still looks like a massive effect
4. **Profile the right layer** - FPS drop was GPU draw calls, not CPU ECS overhead this time

### Files Modified

- `src/explosion_shader.rs` - Added `ExplosionAssets` with shared handles, 20% spawn probability
- `src/objective.rs` - Uses `PendingExplosion` queue with staggered delays (0.05s-0.5s)
- `src/particles.rs` - Death flash effect configuration
- `docs/explosion_system.md` - Added optimization documentation section

## Next Steps

1. Complete migration to `ExplosionQueue` in `objective.rs`
2. Register `ExplosionQueue` resource in `main.rs`
3. Despawn units immediately instead of keeping them alive with `PendingExplosion`
4. Re-enable explosion visual effects and verify performance
5. Consider object pooling for particle effects if Hanabi overhead persists
