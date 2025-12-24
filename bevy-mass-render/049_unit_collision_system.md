# 049: Unit Collision System

## Overview

Implemented M2TW-style mass-based unit collision for 10k units. Units physically push each other when overlapping, with heavier units pushing lighter units more.

## Architecture

### Two-Phase Hybrid System

**Phase 1: Soft Avoidance (Preventive)** - `movement.rs`
- Slows units when approaching neighbors
- Prevents ~90% of collisions before they happen
- Currently disabled (`SOFT_AVOIDANCE_STRENGTH = 0.0`) for performance

**Phase 2: Hard Push (Corrective)** - `collision.rs`
- Detects actual overlaps via spatial grid
- Applies M2TW-style separation vectors
- Mass-based push distribution

### Key Components

| Component | Location | Purpose |
|-----------|----------|---------|
| `UnitMass` | types.rs | Mass for push calculations (default 1.0) |
| `unit_collision_system` | collision.rs | Hard collision resolution |
| Soft avoidance | movement.rs | Speed reduction near neighbors |

### Constants (constants.rs)

```rust
UNIT_COLLISION_RADIUS: f32 = 0.8      // Physical collision radius
SOFT_AVOIDANCE_RADIUS: f32 = 2.0      // Soft avoidance trigger distance
SOFT_AVOIDANCE_STRENGTH: f32 = 0.0    // 0.0 = off, 1.0 = full
COLLISION_PUSH_STRENGTH: f32 = 8.0    // Push force multiplier
DEFAULT_UNIT_MASS: f32 = 1.0          // Droid mass (future: cavalry 5.0)
```

## Algorithm

### Spatial Grid Optimization
- Uses existing `SpatialGrid` (10.0 unit cells, 100x100 grid)
- O(N*k) complexity where k = average neighbors (~45)
- NOT O(N²) - only checks 3x3 neighborhood per unit

### Mass-Based Push (M2TW Style)
```rust
// Heavier unit pushes lighter unit more
let total_mass = mass + other_mass;
let my_ratio = other_mass / total_mass;    // I get pushed by their mass
let their_ratio = mass / total_mass;        // They get pushed by my mass
```

### Query Conflict Resolution
Bevy doesn't allow multiple queries accessing same component. Solved by:
1. Collect all positions into `HashMap<Entity, Vec3>` first
2. Iterate collected data for collision detection
3. Apply pushes via mutable query afterward

## Performance

### Current Status
- FPS dropped from 100 to 60 with soft avoidance enabled
- With soft avoidance disabled: ~80-90 FPS
- Bottleneck: Sequential iteration over 10k units

### Profiled Costs
| Phase | Estimated Time |
|-------|---------------|
| Soft avoidance (enabled) | ~0.3-0.5ms |
| Hard collision | ~0.2-0.4ms |
| HashMap building | ~0.1ms |

## Performance Optimization Plans

| Optimization | Complexity | Expected Gain | Notes |
|--------------|------------|---------------|-------|
| Parallel iteration (`par_iter`) | Low | 2-4x speedup | Use Bevy's parallel query |
| Frame skipping (every 2-3 frames) | Low | 2-3x speedup | Acceptable for collision |
| Smaller grid cells (5.0 instead of 10.0) | Medium | 20-40% fewer checks | More cells, fewer neighbors |
| Temporal caching (reuse pairs) | Medium | 30-50% speedup | Cache collision pairs 2-3 frames |
| GPU compute shader | High | 10x+ for 50k units | Major architecture change |

### Priority Order
1. **Parallel iteration** - Easy win, use `par_iter_mut` ✅ IMPLEMENTED
2. **Frame skipping** - Run collision every 2nd frame
3. **Grid tuning** - Experiment with cell sizes
4. **Caching** - Only if still needed

## Implemented: Parallel Iteration

Added rayon dependency and converted collision detection to parallel:

```rust
// Parallel collision detection - each unit finds its pushes independently
let pushes: Vec<(Entity, Vec3)> = droid_data
    .par_iter()
    .flat_map(|(entity, pos, mass)| {
        let nearby = spatial_grid.get_nearby_droids(*pos);
        let mut local_pushes = Vec::new();
        // ... collision logic ...
        local_pushes
    })
    .collect();
```

Key design decisions:
- Detection phase is embarrassingly parallel (each unit independent)
- Application phase stays sequential (fast, just memory writes)
- Uses `flat_map` pattern for collecting variable-length results

## Files Modified

- `src/types.rs` - Added `UnitMass` component
- `src/constants.rs` - Added collision constants
- `src/movement.rs` - Added soft avoidance to `animate_march`
- `src/collision.rs` - NEW: Hard collision system
- `src/setup.rs` - Added `UnitMass` to droid spawning
- `src/main.rs` - Registered collision module and system

## Future Expansion

Ready for medieval pivot:
- Mass values: Peasant 0.5, Infantry 1.0, Knight 3.0, Cavalry 5.0
- Bracing: Stationary units get +mass
- Momentum: velocity × mass for charge bonus
- Knockdown: Severe mass differential causes stagger
- Crush damage: Units in dense blobs take damage
