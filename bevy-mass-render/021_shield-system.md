# Devlog 021: Shield System Implementation

**Date:** December 1, 2025
**Feature:** Empire at War-style Shield System with Regeneration
**Status:** Complete

## Overview

Implemented a comprehensive shield defense system for uplink towers, inspired by Star Wars: Empire at War. Shields protect towers from laser fire, regenerate after damage, and respawn after destruction - creating strategic depth in tower assaults.

## Features Implemented

### 1. Core Shield Mechanics

**Shield Defense:**
- Hemisphere shield dome protecting uplink towers (50-unit radius)
- Blocks all enemy laser projectiles via sphere collision detection
- Each laser hit deals 25 damage to shield HP (5000 max HP)
- Shield absorbs damage until depleted, then despawns

**Visual Feedback:**
- Custom WGSL shader with hexagonal energy grid pattern
- Fresnel edge glow effect (brighter at grazing angles)
- Animated energy pulse waves across surface
- Health-based alpha fade (more transparent as HP decreases)
- Gradual color shift from team color to white as shield takes damage
- Impact flash effect when hit (0.2s duration, +0.5 alpha boost)

**Team Colors:**
- Team A: Cyan/blue shields `Color::srgb(0.2, 0.6, 1.0)`
- Team B: Orange/red shields `Color::srgb(1.0, 0.4, 0.2)`

### 2. Empire at War-Style Regeneration

**Regeneration System:**
- 3-second delay after last hit before regeneration begins
- 50 HP/second regeneration rate
- Shields regenerate to full HP if not continuously damaged
- Mirrors Empire at War's tactical shield management

**Respawn System:**
- When shield HP reaches 0, shield despawns
- 10-second respawn timer begins
- Shield respawns at **0 HP** (not full HP)
- Gradually regenerates from 0 to full HP (100 seconds total)
- Respawn only occurs if tower is still alive

**Strategic Implications:**
- Attackers must maintain sustained fire to prevent regeneration
- Brief pauses in assault allow shields to recover
- Destroying shields provides ~10-second vulnerability window
- Respawned shields are initially weak, requiring time to strengthen

### 3. Visual Effects

**Ripple System:**
- 25% chance per laser impact to spawn visual ripple
- Up to 8 simultaneous expanding ripple rings
- 1.5-second lifetime per ripple
- Ripples expand at 30 units/second with 4-unit width
- Shader-based rendering for performance

**Particle Effects (Bevy Hanabi):**
- Shield impact particles spawn on 25% of hits
- Cyan energy burst particles (40 particles)
- Particles spawn at shield surface (radius + 0.5 offset)
- 0.8-second particle lifetime
- High drag for quick dispersion

**Audio:**
- Shield impact sound effect (0.4 volume)
- Plays on 25% of hits (synchronized with particles/ripples)

### 4. Refactoring & Code Quality

**ShieldConfig Resource:**
```rust
pub struct ShieldConfig {
    pub max_hp: f32,                   // 5000.0
    pub regen_rate: f32,               // 50.0 HP/s
    pub regen_delay: f32,              // 3.0s delay
    pub respawn_delay: f32,            // 10.0s delay
    pub impact_flash_duration: f32,    // 0.2s
    pub laser_damage: f32,             // 25.0
    pub shield_impact_volume: f32,     // 0.4
    pub surface_offset: f32,           // 0.5
    pub fresnel_power: f32,            // 3.0
    pub hex_scale: f32,                // 8.0
    pub mesh_segments: u32,            // 32
}
```

**Team Color Centralization:**
- Added `Team::shield_color()` method to `types.rs`
- Eliminated color duplication across codebase
- Single source of truth for team visual identity

**Constants & Magic Numbers:**
- Extracted visual alpha constants:
  - `SHIELD_BASE_ALPHA_MIN: 0.3` (minimum at 0 HP)
  - `SHIELD_BASE_ALPHA_MAX: 0.7` (additional at full HP)
  - `SHIELD_IMPACT_FLASH_ALPHA: 0.5` (impact flash boost)
  - `SHIELD_MAX_ALPHA: 0.8` (maximum total)
- Added audio volume constants:
  - `VOLUME_EXPLOSION: 0.5`
  - `VOLUME_LASER: 0.3`
  - `VOLUME_SHIELD_IMPACT: 0.4` (reference only, used via ShieldConfig)

**Helper Functions:**
- `create_shield_material()` - Reduces duplication in shield spawning
- `spawn_shield_with_hp()` - Allows custom starting HP for respawns

**Documentation:**
- Comprehensive doc comments on `Shield` and `DestroyedShield` components
- Clarified ECS state transition model: Active → Destroyed → Respawned

## Technical Implementation

### Shield Component State Machine

```
Active Shield (Shield component)
  ├─ HP > 0: Visible, blocks lasers, regenerates
  └─ HP = 0: Despawn, spawn DestroyedShield marker

DestroyedShield (marker component)
  ├─ Timer counting down (10s)
  ├─ Check tower alive each frame
  └─ Timer = 0: Spawn new Shield at 0 HP

Respawned Shield
  └─ HP = 0: Begin regeneration to full (100s)
```

### System Execution Order

1. **shield_collision_system** - Detect laser hits, apply damage
2. **shield_regeneration_system** - Regenerate HP after delay
3. **shield_impact_flash_system** - Tick down flash timers
4. **shield_health_visual_system** - Update material alpha & ripples
5. **shield_tower_death_system** - Despawn shields when tower dies
6. **shield_respawn_system** - Handle respawn countdown & spawning
7. **animate_shields** - Update shader time for energy pulses

### Shader Architecture

**Vertex Shader:**
- Standard hemisphere mesh (upper half of sphere)
- 32 segments for smooth curvature
- UVs mapped for hexagonal tiling

**Fragment Shader (WGSL):**
- Fresnel calculation: `pow(1.0 - dot(normal, view), 3.0)`
- Hexagonal grid pattern generation
- Sine wave pulse animation: `sin(time * 2.0 + pos.y * 0.5)`
- Ripple expansion/fade calculation (8 simultaneous ripples)
- Health-based color interpolation (team color → white)
- Alpha composition: base + edge + hex pattern + ripple

## Debug Features

**Shield Destruction Shortcut:**
- Press `0` to toggle explosion debug menu
- Press `S` to instantly destroy Team B shield (set HP to 0)
- Useful for testing respawn mechanics without waiting

**Tower HP Increase:**
- Increased tower max HP from 1000 → 2000
- Allows more time to observe shield regeneration behavior
- Makes testing shield respawn easier

## Performance Considerations

**Optimizations:**
- Ripple/particle spawn chance: 25% (prevents audio/visual spam)
- Shader calculations per-pixel (GPU accelerated)
- Particle cleanup via spawn time comparison (no per-frame timer ticking)
- Shield mesh cached in asset system (reused for both teams)

**Impact:**
- Shield systems run efficiently even with particle effects
- No noticeable frame drops during intense shield bombardment
- Ripple system handles 8 simultaneous effects smoothly

## Lessons Learned

1. **ECS State Management:** Using separate components (Shield vs DestroyedShield) is more idiomatic in Bevy than using state enums within a single component. Queries naturally filter by component presence.

2. **Configuration Resources:** Centralizing all tunable values in a Resource makes balancing much easier than scattered constants across files.

3. **Visual Feedback Balance:** 100% particle/sound spawn was overwhelming. 25% chance provides good feedback without spam.

4. **Respawn at Zero:** Respawning shields at 0 HP (not full) creates interesting gameplay - newly respawned shields are vulnerable while regenerating.

5. **Team Color Method:** Adding `Team::shield_color()` method eliminated duplication and makes future team-based features easier (UI, effects, etc).

## Future Improvements

**Potential Enhancements:**
- Directional damage visualization (ripples show attack direction)
- Shield overload effect when taking sustained damage
- Different shield types (fast regen/low HP vs slow regen/high HP)
- Shield power level indicator (visual representation of HP)
- Sound effect pitch variation based on shield health
- Partial shield coverage (hemisphere can be rotated/oriented)

**Balance Tweaks:**
- May need to adjust regen rates based on army DPS
- Respawn delay might need tuning (10s could be too short/long)
- Laser damage per hit (25) may need adjustment

## Related Files

**Core Shield System:**
- `src/shield.rs` - Shield plugin, components, systems, material
- `assets/shaders/shield.wgsl` - Custom WGSL fragment shader
- `src/particles.rs` - Shield impact particle effects
- `src/types.rs` - Team colors and shared types

**Integration:**
- `src/objective.rs` - Tower spawning with shields, debug UI
- `src/combat.rs` - Laser projectiles that shields block
- `src/constants.rs` - Audio volume constants
- `src/main.rs` - System registration and ordering

## Commits

1. **Initial Implementation:**
   - "Implement Empire at War-style shield regeneration and increase tower HP"
   - Shield mechanics, regeneration, respawn, debug shortcuts

2. **Refactoring #1-3:**
   - "Refactor shield system: add config struct, centralize team colors, extract material creation"
   - ShieldConfig resource, Team::shield_color(), create_shield_material()

3. **Refactoring #4-6:**
   - "Refactor shield system: extract magic numbers and add audio volume constants"
   - Visual constants, audio constants, comprehensive documentation

## Testing Checklist

- [x] Shield blocks enemy lasers
- [x] Shield takes damage per laser hit
- [x] Shield regenerates after 3s delay
- [x] Shield despawns at 0 HP
- [x] Shield respawns after 10s
- [x] Respawned shield starts at 0 HP
- [x] Shield does not respawn if tower destroyed
- [x] Active shield despawns when tower dies
- [x] Visual feedback (alpha, color, flash, ripples)
- [x] Particle effects on impact (25% chance)
- [x] Audio effects on impact (25% chance)
- [x] Debug shortcut (0 > S) destroys shield
- [x] Team colors correct (A=cyan, B=orange)
- [x] No friendly fire (shields ignore own team's lasers)

---

**Status:** Feature complete and merged to main branch.
