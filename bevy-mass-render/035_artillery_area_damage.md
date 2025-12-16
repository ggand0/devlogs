# Devlog 035: Artillery Barrage System & Area Damage

**Date:** 2025-12-16
**Branch:** `feat/artillery-area-damage`

## Overview

Added player-controlled artillery barrage system and area damage mechanics for ground explosions. Units in explosion radius now die or get knocked back based on distance from blast center.

## Features Implemented

### Artillery Barrage System

Three firing modes controlled via hotkeys:

| Key | Mode | Behavior |
|-----|------|----------|
| **V** | Single Shot | Click = 1 explosion at cursor (debug/testing) |
| **B** | Scatter Barrage | Click = 6-10 shells scattered around cursor (CoH1-style) |
| **N** | Line Barrage | Drag = shells along a line with red arrow indicator |

- Press key to toggle mode on/off
- Left-click (or drag for N) to fire
- Shells have staggered timing for realistic barrage effect
- Line barrage shows red terrain-conforming arrow during drag

### Area Damage System

Three damage zones (radii scaled by explosion scale parameter):

| Zone | Radius | Effect |
|------|--------|--------|
| Core | 5.0 | Instant death |
| Mid | 12.0 | RNG death (80% at core edge → 20% at mid edge) |
| Rim | 20.0 | Knockback only |

### Death Effects (50/50 Split)

When a unit dies from area damage:
- **50%**: Existing flipbook explosion (`spawn_custom_shader_explosion`)
- **50%**: Ragdoll death - unit flies away in spherical direction, despawns on ground contact

### Knockback Mechanics

Units in rim zone (or mid zone survivors):
- Physics-based push away from explosion center
- Arc trajectory with upward component + gravity
- Lands and enters 1-second stun (no movement/shooting)
- After stun, unit resumes normal behavior

## Technical Details

### New Files

- `src/artillery.rs` - Artillery state resource, input system, visual system, spawn system
- `src/area_damage.rs` - Area damage event processing, knockback/ragdoll physics

### New Components

```rust
// Unit knocked back (survives)
pub struct KnockbackState {
    pub velocity: Vec3,
    pub gravity: f32,
    pub ground_y: f32,
    pub is_airborne: bool,
    pub stun_timer: f32,
}

// Unit dying via ragdoll
pub struct RagdollDeath {
    pub velocity: Vec3,
    pub angular_velocity: Vec3,
    pub gravity: f32,
    pub ground_y: f32,
}

// Event for area damage
pub struct AreaDamageEvent {
    pub position: Vec3,
    pub scale: f32,
}
```

### Constants Added

```rust
// Area damage zones
AREA_DAMAGE_CORE_RADIUS: 5.0
AREA_DAMAGE_MID_RADIUS: 12.0
AREA_DAMAGE_RIM_RADIUS: 20.0

// Knockback physics
KNOCKBACK_BASE_SPEED: 15.0
KNOCKBACK_GRAVITY: -30.0
KNOCKBACK_STUN_DURATION: 1.0

// Ragdoll physics
RAGDOLL_MIN_SPEED: 20.0
RAGDOLL_MAX_SPEED: 35.0
RAGDOLL_GRAVITY: -25.0

// Artillery
ARTILLERY_SCATTER_RADIUS: 25.0
ARTILLERY_SHELL_COUNT_MIN: 6
ARTILLERY_SHELL_COUNT_MAX: 10
ARTILLERY_SHELL_DELAY_MIN: 0.3
ARTILLERY_SHELL_DELAY_MAX: 1.5
ARTILLERY_LINE_MAX_LENGTH: 100.0
ARTILLERY_LINE_SHELL_SPACING: 15.0
```

### System Integration

- `animate_march` excludes units with `KnockbackState` or `RagdollDeath`
- `hitscan_fire_system` excludes units with `KnockbackState` or `RagdollDeath`
- Area damage uses existing `SpatialGrid` for efficient nearby unit queries
- Artillery reuses `create_arrow_mesh` from selection system for line indicator

## Design Notes

### CoH1-style Scatter Barrage
The scatter barrage (B key) mimics Company of Heroes 1 American artillery - multiple shells land in a roughly circular pattern with random scatter and staggered timing. Good for area denial.

### Line Barrage
The line barrage (N key) allows precise placement - useful for hitting a column of units or cutting off retreat paths. Red arrow shows exactly where shells will land.

### Area Damage Philosophy
- Core zone is always lethal (direct hit)
- Mid zone gives survival chance that increases with distance
- Rim zone is purely knockback - units survive but are disrupted
- This creates interesting tactical situations where surviving units in mid zone may be scattered

## Future Improvements

- Visual warning indicator before shell lands (dust cloud, whistling sound)
- Cooldown system for artillery
- Different artillery types (heavier, lighter, different patterns)
- Shell travel time (currently instant impact)
- Crater decals at impact points
