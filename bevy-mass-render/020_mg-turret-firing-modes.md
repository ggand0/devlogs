# Devlog 020: MG Turret Dual Firing Modes Implementation

**Date**: 2025-11-30
**Session Focus**: MG turret rapid-fire system with continuous target switching and audio prioritization

---

## Overview

Implemented a specialized MG (machine gun) turret variant with two distinct firing modes, rapid target acquisition, and custom audio integration. The MG turret provides anti-infantry suppression fire with a characteristic "mowing down" effect.

---

## Design Philosophy

### Realistic Energy Weapon Design
- **Single barrel** instead of gatling mechanism
- Energy weapons don't need barrel rotation for fire rate
- Fire rate limited by capacitor recharge and heat dissipation
- Visible cooling system through periodic pauses
- More reliable than complex multi-barrel systems

### Role Differentiation
- **Heavy Turret** (existing): Anti-vehicle, dual barrels, slow fire rate (2.0s)
- **MG Turret** (new): Anti-infantry, single barrel, rapid fire (0.08s)

---

## Implementation Details

### 1. Firing Mode System

**Component Structure** ([types.rs:324-337](src/types.rs#L324-L337)):
```rust
#[derive(Clone, Copy, PartialEq, Eq)]
pub enum FiringMode {
    Burst,      // Fixed shots then cooldown
    Continuous, // Mowing down with rapid target switching
}

#[derive(Component)]
pub struct MgTurret {
    pub firing_mode: FiringMode,
    pub shots_in_burst: u32,
    pub max_burst_shots: u32,    // 45 shots before pause
    pub cooldown_timer: f32,
    pub cooldown_duration: f32,  // 1.5s pause
}
```

**Mode Behaviors**:
- **Burst Mode**: Fires exactly 40 shots, then 1.0s cooldown
- **Continuous Mode**: Fires 45 shots (~3.6s), then 1.5s cooling pause
  - Immediately switches to new targets when current dies
  - No waiting for standard target acquisition timer
  - Creates suppression fire effect

### 2. Rapid Target Switching

**Implementation** ([combat.rs:503-546](src/combat.rs#L503-L546)):

When in Continuous mode and current target dies:
1. Validates current target is dead
2. Immediately scans for closest enemy in TARGETING_RANGE (150 units)
3. Checks all enemy units first, then enemy towers
4. Assigns new target without interrupting fire cycle
5. Continues burst until max_burst_shots reached

**Key Difference from Standard Targeting**:
- Standard system: Scans every 2.0 seconds (TARGET_SCAN_INTERVAL)
- MG Continuous: Instant retargeting on every frame when target dies

### 3. Audio Integration

**Custom MG Sound** ([setup.rs:70](src/setup.rs#L70)):
- Loaded `mg_1_single.wav` for per-bullet audio
- Volume: 0.25 (reduced from initial 0.4 for balance)

**Audio Prioritization** ([combat.rs:359-363](src/combat.rs#L359-L363)):
```rust
let mut shots_fired = 0;         // Regular units: 5/frame limit
let mut mg_shots_fired = 0;      // MG turret: 3/frame limit
const MAX_AUDIO_PER_FRAME: usize = 5;
const MAX_MG_AUDIO_PER_FRAME: usize = 3;
```

**Benefits**:
- MG sounds never get throttled by regular unit fire
- Separate counter ensures consistent MG audio during bursts
- Prevents audio interruption during rapid fire

### 4. Combat Parameters

**Fire Rate** ([combat.rs:566](src/combat.rs#L566)):
- Interval: 0.08s between shots (12.5 shots/second)
- Laser speed: 3x normal (300 units/sec vs 100 units/sec)
- Rotation speed: 5.0 rad/s (vs 3.0 rad/s for heavy turret)

**Burst Timing**:
- Active phase: 45 shots × 0.08s = ~3.6 seconds
- Cooling phase: 1.5 seconds
- Total cycle: ~5.1 seconds

---

## Technical Challenges Solved

### Challenge 1: Incomplete Bursts
**Problem**: MG stopped firing mid-burst when target died because `current_target` became `None`

**Solution**:
- Check target validity every frame during Continuous mode
- Immediately acquire new target from all_units_query/all_towers_query
- Continue firing without resetting burst counter

### Challenge 2: Audio Interruption
**Problem**: MG sound cut off when many units fired simultaneously (shared 5-sound limit)

**Solution**:
- Separate `mg_shots_fired` counter independent from `shots_fired`
- Dedicated `MAX_MG_AUDIO_PER_FRAME` limit (3 sounds)
- MG audio plays regardless of other unit fire

### Challenge 3: Shot Tracking Across Modes
**Problem**: Burst mode tracked shots but Continuous mode didn't, causing non-stop firing

**Solution**:
- Both modes now track `shots_in_burst`
- Both modes respect `max_burst_shots` and trigger cooldown
- Only difference: Continuous mode rapidly switches targets mid-burst

---

## Code Structure

### New Components
- `FiringMode` enum in [types.rs](src/types.rs)
- Extended `MgTurret` component with burst tracking fields
- `AudioAssets.mg_sound` field

### Modified Systems
- `auto_fire_system`: Mode-specific burst control logic
- `auto_fire_system`: Rapid target acquisition for Continuous mode
- `auto_fire_system`: Separate MG audio counter and volume

### Configuration
- MG spawns in Continuous mode by default ([objective.rs:1484](src/objective.rs#L1484))
- 45-shot bursts with 1.5s cooldown
- Positioned at (10, terrain_height, 10)

---

## Visual/Audio Effect

The MG turret now exhibits:
1. **Rapid muzzle flash**: Visible laser fire every 0.08s
2. **Continuous audio**: Per-bullet sound at 0.25 volume
3. **Target sweeping**: Barrel tracks new enemies instantly
4. **Cooling pauses**: Brief 1.5s silence after 45 shots
5. **High projectile speed**: Lasers reach targets quickly (3x normal)

---

## Future Enhancements

- [ ] Visual muzzle flash effect during active firing
- [ ] Heat distortion shader during bursts
- [ ] Emissive barrel glow that intensifies with shots_in_burst
- [ ] Different colored lasers for MG (yellow/orange vs green)
- [ ] Tracer rounds (every 5th shot brighter)
- [ ] Barrel recoil animation

---

## Files Modified

- `src/types.rs`: Added FiringMode enum, extended MgTurret, added mg_sound
- `src/combat.rs`: Mode-specific firing logic, rapid targeting, audio prioritization
- `src/objective.rs`: MG spawn configuration with Continuous mode
- `src/setup.rs`: MG audio asset loading

---

## Testing Notes

**Verified Behavior**:
- ✅ MG fires 45 shots then pauses for 1.5s
- ✅ Instantly retargets when enemy dies mid-burst
- ✅ Audio plays consistently without interruption
- ✅ Burst counter resets after cooldown
- ✅ Lasers travel 3x faster than normal fire

**Performance**:
- No noticeable FPS impact from rapid target queries
- Audio mixing handles 3 MG sounds/frame smoothly
- Burst tracking adds minimal computational overhead

---

## Debugging Session: Aiming and Hit Detection Issues

**Date**: 2025-12-01

### Initial Problem Report
User observed MG turret "wasting rounds" - firing multiple shots at the same target without hits. Initially suspected despawn lag, but investigation revealed the actual root causes were aiming bugs.

### Investigation Process

1. **Hypothesis 1: Despawn Lag** ❌
   - Checked unit death/despawn timing in `collision_detection_system`
   - Found units despawn **immediately** when hit (no Health component)
   - Ruled out as cause

2. **Hypothesis 2: Projectiles In-Flight** ⚠️
   - At 0.05s fire rate and ~0.17s travel time, MG fires 3-4 shots before first hits
   - Contributes to wasted shots but not the main issue
   - This is inherent to projectile-based combat (vs hitscan)

3. **Root Cause 1: Turret Rotation Bug** ✅
   - **Problem**: `Transform::IDENTITY.looking_at(direction_vector)` is incorrect
   - This was causing consistent aiming errors - turret pointing wrong direction
   - **Fix**: Changed to `Transform::from_translation(turret_pos).looking_at(target_pos_flat, Vec3::Y)`
   - Now properly orients turret's -Z axis toward target

4. **Root Cause 2: Height Offset Mismatch** ✅
   - **Problem**: Firing logic aimed at `target_pos + Vec3::new(0.0, 0.8, 0.0)`
   - Collision sphere centered at **ground level** (Y=0.0 relative to unit)
   - Lasers shooting 0.8 units **above** the collision sphere
   - **Fix**: Removed Y offset, now aims at `target_pos` (collision sphere center)

### Debug Visualization Tool

Implemented collision sphere visualization to diagnose the issue:
- **Wireframe spheres** overlaid on units and buildings
- Green spheres for units (radius 1.0), orange for buildings (radius 3.0-4.0)
- Keyboard toggle: Press `0` for debug menu, then `C` to toggle visualization
- System: `visualize_collision_spheres_system` ([combat.rs:852-910](src/combat.rs#L852-L910))
- Created `ExplosionDebugMode.show_collision_spheres` flag
- Visualization spawns/despawns each frame for real-time feedback

**Implementation Details**:
- Latitude/longitude wireframe mesh with 12-16 segments
- Semi-transparent material (alpha 0.3) for see-through effect
- `DebugCollisionSphere` marker component for cleanup
- Integrated with existing debug mode UI

### Projectile Speed Tuning

**Attempted Fix: Increase Laser Speed to 8x** ❌
- Goal: Make projectiles hit before next shot fires
- At 8x speed (800 units/sec), projectiles move 13.3 units per frame (60 FPS)
- **New Problem**: Projectile tunneling through 1.0 radius collision spheres
- Collision detection runs per-frame; fast projectiles skip past targets

**Final Solution: Revert to 3x Speed** ✅
- At 3x speed (300 units/sec), projectiles move 5 units per frame
- Provides better collision detection reliability
- Still faster than standard fire for MG feel
- Some wasted shots acceptable (realistic behavior)

### Known Issues & Future Work

**Current Limitations**:
- Multiple projectiles in-flight still cause 2-3 wasted shots per target kill
- This is inherent to projectile-based combat system
- At 0.05s fire interval, unavoidable with current architecture

**Planned Enhancement** (not this branch):
- Implement **hitscan detection system** for instant hit registration
- Would eliminate all wasted shots
- Trade-off: Loses visible projectile travel for gameplay feedback
- Hybrid approach possible: hitscan for hit detection, visual projectiles for effect

### Fixes Applied

**File: `src/combat.rs`**

1. **Turret Rotation Fix** (lines 769-779):
```rust
// BEFORE (WRONG):
let direction = (target_pos - turret_pos).normalize();
let target_rotation = Transform::IDENTITY
    .looking_at(Vec3::new(direction.x, 0.0, direction.z), Vec3::Y);

// AFTER (CORRECT):
let target_pos_flat = Vec3::new(target_pos.x, turret_pos.y, target_pos.z);
let target_rotation = Transform::from_translation(turret_pos)
    .looking_at(target_pos_flat, Vec3::Y);
```

2. **Aiming Height Fix** (lines 597, 406):
```rust
// BEFORE: Aim 0.8 units above ground
let target_pos = target_transform.translation() + Vec3::new(0.0, 0.8, 0.0);

// AFTER: Aim at collision sphere center (ground level)
let target_pos = target_transform.translation();
```

3. **Laser Speed** (line 571):
```rust
// Final value: 3x speed for reliable collision detection
(&mg_barrel_positions[..], 0.05, LASER_SPEED * 3.0, LASER_LENGTH * 0.6)
```

**File: `src/objective.rs`**
- Extended `ExplosionDebugMode` resource with `show_collision_spheres: bool` field
- Added `C` key toggle in `debug_warfx_test_system`
- Updated debug UI text to show collision sphere toggle option

**File: `src/main.rs`**
- Registered `visualize_collision_spheres_system` in Update schedule

### Testing Results

**Before Fixes**:
- Turret consistently missed targets by ~0.8 units vertically
- Lasers visibly passing above units
- 8-10 shots per kill (mostly misses)

**After Fixes**:
- Turret aims directly at collision sphere center
- Hit rate improved significantly
- 2-3 shots per kill (only in-flight projectiles wasted)
- Acceptable for projectile-based combat

### Debug Tool Value

The collision sphere visualization proved **critical** for diagnosing aiming issues:
- Made invisible collision boundaries visible
- Revealed height offset mismatch immediately
- Useful for future gameplay tuning and balancing
- Kept as permanent debug feature (toggle with `C` key)

---

## Lessons Learned

1. **Query Optimization**: Added specific queries (`all_units_query`, `all_towers_query`) rather than iterating combat_query to avoid borrow conflicts
2. **Audio Design**: Separate audio limits critical for weapon identity and player feedback
3. **Mode Flexibility**: Single component supporting multiple behaviors better than separate turret types
4. **Timing Balance**: 45-shot bursts feel substantial without being overwhelming
5. **Volume Tuning**: 0.25 volume provides presence without drowning out other game audio
6. **Transform Math**: `Transform::IDENTITY.looking_at(direction)` is wrong; use `Transform::from_translation(pos).looking_at(target)` for proper orientation
7. **Visual Debugging**: Collision sphere visualization essential for diagnosing aiming issues - invest in debug tools early
8. **Projectile Physics**: Fast projectiles (>10 units/frame) cause tunneling through small collision spheres; balance speed vs detection reliability
9. **Aim Points**: Always match aiming logic to collision detection points - don't assume visual mesh center equals collision center
