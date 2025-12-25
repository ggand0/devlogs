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
- Uses existing `SpatialGrid` (5.0 unit cells, 200x200 grid)
- O(N*k) complexity where k = average neighbors (~10-15)
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
1. **Parallel iteration** ✅ IMPLEMENTED - rayon `par_iter`
2. **Frame skipping** ✅ IMPLEMENTED - runs every 2nd frame
3. **Grid tuning** ✅ IMPLEMENTED - 5.0 cells, 200x200 grid
4. **Stationary culling** ✅ IMPLEMENTED - skip idle units
5. **Temporal caching** ❌ SKIPPED - conflicts with frame skipping (see below)

## Implemented Optimizations

### 1. Parallel Iteration

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

### 2. Frame Skipping

```rust
const COLLISION_FRAME_SKIP: u32 = 2;

// Compensate delta for skipped frames
let delta = time.delta_secs() * COLLISION_FRAME_SKIP as f32;
```

At 60+ FPS, collision updates at 30+ Hz - imperceptible for push physics.

### 3. Grid Tuning

Changed from 10.0 to 5.0 unit cells:
- `GRID_CELL_SIZE: 5.0` (was 10.0)
- `GRID_SIZE: 200` (was 100)
- Memory: ~560KB → ~2MB (trivial)
- Fewer neighbors per cell = fewer distance checks

### 4. Stationary Unit Culling

Only moving units initiate collision checks:
```rust
let moving_droids = droids.iter().filter(|d| {
    target_spawn_dist > STATIONARY_THRESHOLD || droid.returning_to_spawn
});
```

Stationary units still get pushed by moving neighbors.

## Decision: Temporal Caching Skipped

**Problem:** Frame skipping and temporal caching are redundant - both save work by using stale data.

| Approach | Staleness |
|----------|-----------|
| Frame skip alone | 2 frames |
| Temporal cache alone | 2-3 frames |
| Both combined | 4-6 frames (multiplicative!) |

At 60 FPS, 6-frame staleness = 100ms delay. Units moving at 3 units/sec travel 0.3 units - visible clipping.

**Decision:** Keep frame skipping, skip temporal caching. Simpler and sufficient.

## GPU Compute Feasibility Evaluation

### When to Consider
- 50k+ units where CPU becomes bottleneck
- Need sub-millisecond collision for other systems

### Current CPU Data Flow
```
ECS Query → collect Vec<(Entity, Vec3, f32)> → build HashMap
→ spatial_grid.get_nearby_droids() → collision math → Vec<(Entity, Vec3)> pushes
→ apply to Transform
```

### GPU Implementation Approach

**1. Data Buffers (upload each frame)**
```rust
// Pack position + mass into Vec4 for GPU
struct GpuUnitData {
    positions: StorageBuffer<[Vec4; MAX_UNITS]>,  // xyz = pos, w = mass
    push_results: StorageBuffer<[Vec4; MAX_UNITS]>,  // xyz = push, w = unused
    grid_cell_counts: StorageBuffer<[u32; GRID_SIZE * GRID_SIZE]>,
    grid_entries: StorageBuffer<[u32; MAX_UNITS]>,  // sorted by cell
}
```

**2. Compute Shader (WGSL)**
```wgsl
@compute @workgroup_size(256)
fn collision_detect(@builtin(global_invocation_id) id: vec3<u32>) {
    let unit_idx = id.x;
    if (unit_idx >= uniforms.unit_count) { return; }

    let pos = positions[unit_idx];
    let my_mass = pos.w;
    let grid_cell = world_to_grid(pos.xyz);

    var push = vec3(0.0);

    // Check 3x3 neighborhood
    for (var dx: i32 = -1; dx <= 1; dx++) {
        for (var dz: i32 = -1; dz <= 1; dz++) {
            let cell_idx = (grid_cell.x + dx) + (grid_cell.z + dz) * GRID_SIZE;
            let cell_start = grid_offsets[cell_idx];
            let cell_end = grid_offsets[cell_idx + 1];

            for (var i = cell_start; i < cell_end; i++) {
                let other_idx = grid_entries[i];
                if (other_idx <= unit_idx) { continue; }  // avoid double processing

                let other_pos = positions[other_idx];
                let dx = pos.x - other_pos.x;
                let dz = pos.z - other_pos.z;
                let dist_sq = dx * dx + dz * dz;

                if (dist_sq < COLLISION_DIST_SQ && dist_sq > 0.0001) {
                    let dist = sqrt(dist_sq);
                    let overlap = COLLISION_DIST - dist;
                    let push_dir = vec3(dx / dist, 0.0, dz / dist);

                    let total_mass = my_mass + other_pos.w;
                    let my_ratio = other_pos.w / total_mass;

                    push += push_dir * overlap * PUSH_STRENGTH * my_ratio;
                }
            }
        }
    }

    push_results[unit_idx] = vec4(push, 0.0);
}
```

**3. The Hard Part: GPU Spatial Grid**

Current CPU grid uses `Vec<Vec<Vec<Entity>>>` - can't use on GPU. Options:

| Approach | Complexity | Performance |
|----------|------------|-------------|
| **Counting sort** | Medium | Fast - O(N) |
| Atomic append | Low | Slower - contention |
| Hash table | High | Complex synchronization |

**Counting sort approach (recommended):**
1. Pass 1: Count units per cell → `grid_cell_counts[]`
2. Prefix sum → `grid_offsets[]` (where each cell starts)
3. Pass 2: Write unit indices to `grid_entries[]` at offsets

**4. Bevy Integration**
```rust
// In render world, run before main pass
fn prepare_collision_buffers(
    query: Query<(Entity, &Transform, Option<&UnitMass>), With<BattleDroid>>,
    mut gpu_data: ResMut<GpuCollisionData>,
) {
    // Upload positions to GPU buffer
    let data: Vec<Vec4> = query.iter()
        .map(|(_, t, m)| Vec4::new(t.translation.x, t.translation.y, t.translation.z,
                                    m.map(|m| m.0).unwrap_or(1.0)))
        .collect();
    gpu_data.positions.write(&data);
}

fn readback_push_results(
    mut gpu_data: ResMut<GpuCollisionData>,
    mut query: Query<&mut Transform, With<BattleDroid>>,
) {
    // Download push results (async readback for no stall)
    let pushes = gpu_data.push_results.read();
    // Apply to transforms...
}
```

### Challenges

| Challenge | Difficulty | Notes |
|-----------|------------|-------|
| GPU spatial grid build | **Hard** | Need counting sort or atomics |
| Readback latency | Medium | Use async readback (1 frame delay) |
| Entity↔Index mapping | Medium | Need stable ordering |
| Bevy render graph | Medium | Custom render node |

### Architecture Decision

**Option A: Full GPU** - Grid + collision + push all on GPU
- Conflicts with CPU movement system
- Would need GPU-side transforms

**Option B: Hybrid (recommended)** - GPU detection, CPU application
- Upload positions each frame (~80KB for 10k units)
- Run compute shader
- Readback pushes (~80KB)
- Apply on CPU (integrates with ECS)

### Estimated Effort
- GPU spatial grid: 8-12 hours (hardest part)
- Compute shader: 4-6 hours
- Bevy integration: 6-10 hours
- **Total: 18-28 hours**

### Verdict
**Not needed yet.** Current CPU handles 10k at 80-90 FPS. GPU becomes worthwhile at 50k+ units where:
- CPU can't keep up even with all optimizations
- Need collision to take <1ms for other systems

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
