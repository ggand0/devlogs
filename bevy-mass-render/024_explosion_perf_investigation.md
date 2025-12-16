# Explosion Performance Investigation - December 3, 2025

## 1. Problem Statement
Tower explosions cause severe FPS drops from 120 FPS to 20-30 FPS. When a tower is destroyed with 100+ units in range, the game becomes unplayable for several seconds.

## 2. Investigation Methodology

### 2.1. Initial Hypothesis (WRONG)
**Theory**: GPU draw call overhead from per-entity material instancing.

**Assumption**: Each `ExplosionMaterial.add()` creates a unique material = 1000+ draw calls.

### 2.2. Ablation Test Results

| Test | Configuration | Result | Conclusion |
|------|---------------|--------|------------|
| 1 | Billboards + Particles enabled | 30-50 FPS | Baseline problem |
| 2 | Billboards disabled, Particles only | 30-50 FPS | Billboards NOT the cause |
| 3 | Particles disabled, Billboards only | ~100 FPS | Particles add ~50 FPS drop |
| 4 | **BOTH disabled (zero visuals)** | **~80 FPS** | **Something else is the bottleneck** |

### 2.3. Critical Discovery
**Test 4 was the key insight**: Even with ZERO explosion visual effects spawned, FPS still dropped to 80. This proved the bottleneck was NOT rendering-related.

## 3. Root Cause Analysis

### 3.1. The ACTUAL Bottleneck: ECS Query Overhead

The `pending_explosion_system` queries ALL pending explosion entities every frame:

```rust
// O(n) every frame where n = 700+ entities
for (entity, mut pending, transform, tower_component) in explosion_query.iter_mut() {
    pending.delay_timer -= time.delta_secs();  // Tick every entity
    if pending.delay_timer <= 0.0 {
        // Process explosion
    }
}
```

### 3.2. Why 700+ Entities Accumulate

With `EXPLOSION_DELAY_MAX = 2.0` seconds:
- 1000 units die at t=0, each gets `PendingExplosion` component
- At t=0.5s: ~750 entities still pending (queried every frame)
- At t=1.0s: ~500 entities still pending
- At t=1.5s: ~250 entities still pending
- At t=2.0s: 0 entities pending

**Peak load**: 700-800 `PendingExplosion` entities queried every single frame for 2 seconds.

### 3.3. Log Evidence

```
EXPLOSION FRAME: 799 total pending, 35 units ready, 0 towers ready
EXPLOSION FRAME: 779 total pending, 15 units ready, 0 towers ready
EXPLOSION FRAME: 764 total pending, 0 units ready, 0 towers ready
```

The `total_pending` count stays at 700+ for the entire delay window.

### 3.4. Secondary Bottleneck: Hanabi GPU Overhead

When particles ARE enabled, there's a secondary GPU bottleneck:

| Entity Count | FPS | Notes |
|--------------|-----|-------|
| 650 | 40-50 | Original (2 effects/unit) |
| 317 | 69-95 | After reducing to 1 effect/unit |
| 100 | 150 | Low entity count |
| 30 | 130 | Minimal overhead |

Each `ParticleEffect` entity has ~0.1ms GPU overhead (compute dispatch + draw call).

## 4. Conclusions (CORRECTED)

### 4.1. Primary Bottleneck
**ECS query overhead** from iterating 700+ `PendingExplosion` component entities every frame.

### 4.2. Secondary Bottleneck
**Hanabi per-entity GPU overhead** - each ParticleEffect entity adds ~0.1ms GPU time.

### 4.3. NOT Bottlenecks (Correcting Previous Analysis)
1. ~~GPU draw calls from per-entity materials~~ - This was NOT the issue
2. ~~Material instancing breaking batching~~ - Irrelevant since visuals weren't the cause
3. ~~Transparent material overhead~~ - Not the primary issue
4. CPU animation systems are efficient (0.2ms for 1000 entities)

## 5. Solution Implemented

### 5.1. Quick Mitigation: Reduce Delay Window
```rust
// Before
pub const EXPLOSION_DELAY_MAX: f32 = 2.0;

// After
pub const EXPLOSION_DELAY_MAX: f32 = 0.5;
```
Reduces max pending entities from ~700 to ~175.

### 5.2. Proper Fix: Centralized Explosion Queue

Replace per-entity components with a single Resource:

```rust
#[derive(Resource, Default)]
pub struct ExplosionQueue {
    pub queue: Vec<ScheduledExplosion>,  // Sorted by trigger_time
}

pub struct ScheduledExplosion {
    pub trigger_time: f64,
    pub position: Vec3,
    pub radius: f32,
    pub is_tower: bool,
}
```

**Performance comparison**:
- Old: O(n) per frame - query 700 entities every frame
- New: O(k) per frame - only process k ready explosions (typically 20-50)

### 5.3. Hanabi Optimization
- Reduced buffer sizes: 32768 → 64-128
- Single effect per unit (sparks only, removed debris)
- Reduced particle spawn counts

## 6. Lessons Learned

1. **Always do ablation testing** - Disabling ALL visuals revealed the true bottleneck
2. **ECS queries have overhead** - Iterating thousands of entities per frame adds up
3. **CPU time vs Frame time** - Low CPU time with high frame time suggests GPU bottleneck, but in this case even the "GPU bottleneck" was masking an ECS issue
4. **Check entity counts** - The "700 total pending" in logs was the smoking gun
5. **Don't assume** - Initial GPU/material hypothesis was completely wrong

## 7. Data Sources
- Log files: `logs/explosion_debug_120325_*.log`
- Test date: December 3, 2025
- Build profile: `opt-dev`
- GPU: AMD Radeon (via Vulkan)
- Baseline FPS: 120 (no explosions)
- Peak pending entity count: 799

## 8. Related Documents
- [025_explosion_ecs_bottleneck_root_cause.md](./025_explosion_ecs_bottleneck_root_cause.md) - Detailed root cause analysis
