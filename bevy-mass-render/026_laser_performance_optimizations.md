# 026: Laser System Performance Optimizations

**Date:** 2025-12-04
**Status:** Partially Complete

## Summary

Investigation and optimization of lag spikes caused by MG turret rapid-fire (12.5 shots/sec). Focused on reducing per-frame and per-shot allocations.

## Optimizations Applied

### 1. Cache Laser Materials and Meshes (Committed)

**Problem:** Each laser shot allocated new `Handle<Mesh>` and `Handle<StandardMaterial>`, causing memory pressure with rapid-fire weapons.

**Solution:** Created `LaserAssets` resource initialized at startup:

```rust
#[derive(Resource)]
pub struct LaserAssets {
    pub team_a_material: Handle<StandardMaterial>,
    pub team_b_material: Handle<StandardMaterial>,
    pub laser_mesh: Handle<Mesh>,
    pub mg_laser_mesh: Handle<Mesh>,
}
```

**Files:** `src/types.rs`, `src/combat.rs`, `src/main.rs`

**Commit:** `a4d57ec` - "Cache laser materials and meshes to reduce per-shot allocations"

### 2. Spatial Grid Throttle (Reverted)

**Problem:** Spatial grid rebuilt every frame with 10,000 droid insertions.

**Attempted Solution:** Only rebuild every 6 frames since units move slowly (3 units/sec) relative to 10-unit grid cells.

**Outcome:** Caused rear squads to stop firing - stale grid data meant collision detection failed for lasers traveling through outdated positions.

**Commit:** `c1e6111` - Reverted

**Future Fix:** Implement hit-scan collision instead of projectile-based. Hit-scan eliminates need for spatial grid entirely - instant raycast at fire time, no per-frame collision checks.

### 3. Periodic Lag Spikes (5-6 seconds)

**Symptoms:** Regular lag spikes even with 100 units (1 squad per team).

**Investigation:** Initially suspected per-frame `Vec` allocations in `target_acquisition_system`. Implemented `TargetAcquisitionCache` to reuse vectors.

**Outcome:** Lag interval increased from 5-6s to 9-10s but persisted. Further investigation revealed lag correlated with `NetworkManager` process activity. Issue resolved after system reboot - likely OS-level memory management or driver state.

**Conclusion:** Not a game code issue.

## Potential Future Optimizations

### Vector Caching for Target Acquisition

The `target_acquisition_system` allocates vectors every frame:

```rust
// Before: allocates ~240KB per frame with 10k units
let all_units: Vec<(Entity, Vec3, Team)> = combat_query.iter().collect();

// After: reuses cached vectors
cache.all_units.clear();
cache.all_units.extend(combat_query.iter()...);
```

Implementation exists in stash but not yet committed. Consider applying when hit-scan is implemented.

## Lessons Learned

1. **Throttling shared state is risky** - The spatial grid is used for collision detection where timing matters. Stale data breaks functionality.

2. **Profile before optimizing** - The 5-6 second lag spikes were OS-level, not game code. Would have wasted time optimizing the wrong thing.

3. **Hit-scan > Projectiles for performance** - Projectile-based collision requires per-frame spatial queries for all active projectiles. Hit-scan is O(1) per shot.
