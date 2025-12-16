# Devlog 029: Hitscan Weapon System

**Date:** 2025-12-08
**Branch:** `feat/hitscan-weapon`

---

## Implemented

### Hitscan Weapons for Infantry
- Replaced projectile-based infantry weapons with instant hitscan
- Damage is applied instantly via raycast; visual tracer is cosmetic only
- Turrets still use traditional projectiles

### Changes Made
1. **`src/types.rs`**
   - Added `WeaponType` enum (for future weapon swapping)
   - Added `HitscanTracer` component for visual effect
   - Added `team` field to `TurretBase` for targeting

2. **`src/constants.rs`**
   - `HITSCAN_TRACER_SPEED`: 400.0
   - `HITSCAN_TRACER_LENGTH`: 4.0
   - `HITSCAN_TRACER_WIDTH`: 0.25

3. **`src/combat.rs`**
   - New `hitscan_fire_system` - handles infantry firing
   - New `update_hitscan_tracers` - moves visual tracers
   - New `perform_hitscan` - ray-sphere intersection for hit detection
   - Modified `auto_fire_system` - now only handles turrets
   - Added turrets to `target_acquisition_system`

4. **`src/turrets.rs`**
   - Added `team: Team::A` to `TurretBase` spawning

---

## Bug Fix: Turrets Now Take Damage from Infantry

### Original Issue
Visual tracers correctly traveled from infantry TO the turret, but turret health bar didn't decrease.

### Root Cause
Turrets use a parent-child entity hierarchy:
- **Parent:** `TurretBase` (has `Health`, `BuildingCollider`)
- **Child:** `TurretRotatingAssembly` (has `BattleDroid`, `CombatUnit`)

Infantry target the **child** entity (because it has `BattleDroid` for targeting). But the `Health` component is on the **parent** entity.

The `HitTower` damage code was calling `turret_query.get_mut(target_entity)` where `target_entity` was the child, not the parent.

### Solution
Added a `ChildOf` query to look up the parent entity when the target is a turret assembly:

```rust
turret_assembly_query: Query<&ChildOf, With<TurretRotatingAssembly>>,

// In HitTower damage case:
if let Ok(child_of) = turret_assembly_query.get(target_entity) {
    let parent_entity = child_of.parent();
    if let Ok((_, mut turret_health)) = turret_query.get_mut(parent_entity) {
        turret_health.damage(25.0);
    }
}
```

### Result
Turrets now correctly take damage from infantry hitscan weapons.

---

## Commits
- `a3bd43b` - Initial hitscan implementation (unit-to-unit works)
