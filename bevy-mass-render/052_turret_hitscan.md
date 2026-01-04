# 052: Turret Hitscan Conversion

## Overview

Converted turrets from projectile-based weapons to instant hitscan, matching infantry behavior. This eliminates "wasted shots" when targets die before projectiles arrive.

## Problem

With projectile-based turrets (especially MG at 20 shots/sec):
- Projectiles take ~0.17s travel time
- 3-4 shots are in-flight before first hit lands
- When target dies, remaining projectiles hit nothing
- Felt wasteful and frustrating to watch

## Solution

Use hitscan (instant raycast) for damage calculation, spawn visual-only tracers for feedback.

## Implementation

### New System: `turret_hitscan_fire_system()` (combat.rs:947-1280)

Queries `TurretRotatingAssembly` entities and:

1. **MG Firing Mode Control** - Preserves burst/continuous mechanics
   - Burst: 45 shots then 1.5s cooldown
   - Continuous: Same, but instantly switches targets when current dies

2. **Target Validation** - Check if target still exists before firing

3. **Barrel Position Calculation**
   ```rust
   let local_barrel_pos = barrel_positions[turret.current_barrel_index % barrel_positions.len()];
   let world_barrel_offset = local_transform.rotation * local_barrel_pos;
   let firing_pos = global_transform.translation() + world_barrel_offset;
   ```
   - MG: `[0.0, 2.0, -7.4]` (single center barrel)
   - Heavy: `[-1.8, 1.5, -6.0]`, `[1.8, 1.5, -6.0]` (alternating)

4. **Accuracy Check** - Uses `calculate_hit_chance()` with `TURRET_BASE_ACCURACY` (80%)
   ```rust
   let hit_chance = calculate_hit_chance(
       TURRET_BASE_ACCURACY,  // 0.80
       firing_pos,
       target_pos,
       true,                  // Turrets always stationary
       target_stationary,
   );
   let hit_success = rand::random::<f32>() < hit_chance;
   ```

5. **Impact Position Determination**
   - Miss: Tracer goes past target with random offset
   - Hit: Check shield intersection first, then raycast for unit collision

6. **Shield Intersection** - Ray-sphere test against enemy shields
   ```rust
   if let Some((hit_dist, hit_pos)) = ray_sphere_intersection(
       firing_pos, direction, shield.center, shield.radius
   ) {
       // Damage shield, tracer stops at hit_pos
   }
   ```

7. **Instant Hit Detection** - Reuses `perform_hitscan()` from infantry
   - `HitUnit` → Despawn target, remove from squad
   - `HitTower` → Apply `HITSCAN_DAMAGE` to building health
   - `Miss` → Tracer continues to target position

8. **Visual Tracer Spawn**
   ```rust
   commands.spawn((
       Mesh3d(tracer_mesh),
       MeshMaterial3d(laser_material),
       Transform::from_translation(firing_pos).with_rotation(tracer_rotation),
       HitscanTracer {
           start_pos: firing_pos,
           end_pos: impact_pos,
           progress: 0.0,
           speed: tracer_speed,  // MG: 600, Heavy: 500
           team: droid.team,
       },
   ));
   ```

9. **Audio** - Proximity-based volume with separate MG/Heavy sounds

### System Registration (main.rs)

```rust
// Before
hitscan_fire_system,      // Infantry
auto_fire_system,         // Turrets (projectiles)

// After
hitscan_fire_system,          // Infantry
turret_hitscan_fire_system,   // Turrets (hitscan now)
```

### Deprecated: `auto_fire_system()`

Marked with `#[allow(dead_code)]` and kept for reference.

## Accuracy Modifiers (Still Apply)

| Modifier | Effect |
|----------|--------|
| Base | 80% (vs infantry 70%) |
| High ground | +10% if 3+ units above target |
| Target moving | -10% if target is moving |
| Range falloff | 0-100u: 0%, 100-150u: -20%, 150-200u: -40% |

Stationary shooter bonus (+15%) does NOT apply - turrets are always stationary and already have higher base accuracy.

## Tracer Speeds

| Turret | Speed | Mesh |
|--------|-------|------|
| MG | 600 units/sec | `mg_laser_mesh` (60% length) |
| Heavy | 500 units/sec | `hitscan_tracer_mesh` |

Both faster than projectiles (100 units/sec) for instant visual feedback.

## What's Preserved

- MG burst/cooldown mechanics
- MG continuous mode target switching
- Barrel cycling for Heavy turret
- Fire rate intervals (MG: 0.05s, Heavy: 2.0s)
- Turret rotation system (separate)
- Audio with proximity-based volume
- Shield interaction
- Line of sight checks

## Files Modified

| File | Changes |
|------|---------|
| `src/combat.rs` | +341 lines: `turret_hitscan_fire_system()`, deprecated `auto_fire_system()` |
| `src/main.rs` | Replaced system registration |

## Result

- No more wasted shots on dead targets
- Every successful accuracy roll = instant damage
- Visual tracers still provide feedback
- Much less frustrating to watch MG turret in action
