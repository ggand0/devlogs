# Devlog 028: Firebase Delta Scenario - Complete Implementation

**Date:** 2025-12-06 (updated 2025-12-08)
**Branch:** feature/firebase-delta-scenario

## Overview

Completed the Firebase Delta hilltop defense scenario with full wave-based gameplay, two-tier strategic/tactical wave system, continuous spawning, preparation phase, turret placement, reinforcements, health bars, visual effects, and terrain decorations.

---

## Features Implemented

### 1. Two-Tier Wave System (Strategic + Tactical)

**Files:** [src/scenario/](src/scenario/) (refactored into submodules)

Implemented a two-tier wave system for continuous combat flow:

**Strategic Waves (Assaults):**
- 3 strategic waves representing major assault phases
- 30-second cooldown between strategic waves
- Player reinforcements spawn during strategic cooldown

**Tactical Waves:**
- 3 tactical waves per strategic wave (9 total)
- Continuous spawning - next wave starts 8 seconds after previous finishes spawning
- Waves overlap: enemies from wave N still fighting while wave N+1 spawns
- Escalating difficulty: 100 → 150 → 200 enemies per tactical wave

**Wave State Machine:**
```
Idle → Preparation → Combat → StrategicCooldown → Combat → ... → Complete
```

**Key Design:** No "dead time" between waves. The battlefield always has action as waves blend together during combat.

### 2. Reinforcement System

**File:** [src/scenario/wave.rs](src/scenario/wave.rs)

- Player reinforcements spawn from south during strategic cooldown
- 2 squads (100 units) arrive after each assault is cleared
- Blue spawn point marker at south edge of map
- Reinforcements are friendly Team A units under player control

### 3. Spawn Point Markers

**File:** [src/scenario/mod.rs](src/scenario/mod.rs)

Visual markers for spawn locations:
- **Enemy spawns (North, East):** Red poles with orange glowing beacons
- **Friendly spawn (South):** Blue pole with cyan glowing beacon
- Emissive materials for visibility
- `NotShadowCaster` to prevent ground shadow artifacts

### 4. Preparation Phase

**Files:** [src/scenario/placement.rs](src/scenario/placement.rs), [selection/mod.rs](src/selection/mod.rs)

- 60-second preparation phase before wave 1 begins
- **Turret Placement:** Left-click to place turrets during prep phase
- **Undo Support:** Right-click to remove last placed turret
- Ghost preview of turret placement position
- Countdown timer UI shows remaining prep time
- Disabled squad selection during prep phase to prevent interference

### 5. Turret Health & Death System

**Files:** [turrets.rs](src/turrets.rs), [combat.rs](src/combat.rs)

- Turrets now take damage from enemy fire (collision detection integration)
- `TurretHealth` component with configurable max HP (500 for MG, 800 for Heavy)
- Death system triggers explosion effects when health depletes
- Turrets despawn after death with visual/audio feedback

### 6. Shader-Based Health Bars

**Files:** [turrets.rs](src/turrets.rs), [objective.rs](src/objective.rs), [assets/shaders/](assets/shaders/)

Created GPU-accelerated health bar system using custom WGSL shaders:

**Turret Health Bars:**
- Green bar that shrinks from right to left as damage accumulates
- Billboard-style rendering (always faces camera)
- Positioned above turret models

**Tower Health Bars:**
- Same shader-based system for command bunker/towers
- Larger scale appropriate for tower size

**Shield Health Bars:**
- Cyan-colored variant for shield generators
- Separate shader (`shield_bar.wgsl`) with appropriate color scheme
- Shows shield integrity percentage

**Technical Details:**
- `HealthBarMaterial` and `ShieldBarMaterial` custom materials
- Camera-relative positioning to prevent overlap issues
- Fixed shrink direction bug caused by billboard UV mirroring

### 7. Turret WFX Explosions

**Files:** [wfx_spawn.rs](src/wfx_spawn.rs), [particles.rs](src/particles.rs)

Lighter explosion effect specifically tuned for turret destruction:

| Component | Tower Explosion | Turret Explosion |
|-----------|-----------------|------------------|
| Center glows | 2 | 1 |
| Flame particles | 57 | 28 |
| Glow sparkles | 35 | 5 |
| Dot sparkles | 75 | 50 |
| Smoke emitter | Yes | No |
| Lifetime | 0.7s | 0.4-0.5s |
| Scale | 4.0x | 1.0-1.5x |

Debug hotkey: Press `7` to test turret explosion at map center.

### 8. Proximity-Based Audio

**File:** [constants.rs](src/constants.rs)

Implemented distance-based volume attenuation for spatial audio:

```rust
pub fn proximity_volume(distance: f32, max_volume: f32) -> f32
```

- Full volume within 100 units
- Linear falloff to minimum (0.005) at 400 units
- Applied to turret firing, explosions, and shield impacts
- Tuned for RTS camera heights (150-200 units)

**Volume Constants:**
- `VOLUME_MG_TURRET: 0.02` (reduced from previous)
- `VOLUME_HEAVY_TURRET: 0.015`
- `VOLUME_TURRET_EXPLOSION: 0.15`

### 9. Procedural Rock Decorations

**File:** [terrain_decor.rs](src/terrain_decor.rs) (new - 491 lines)

Desert terrain visual enhancement with scattered rocks:

**Rock Mesh Generation:**
- Procedural deformed icosahedron using Perlin noise displacement
- 5 unique mesh variants generated at spawn time
- Variable scale (0.5 - 3.5x), rotation, and color variation

**Placement Algorithm:**
- 150 rocks per map (configurable via `ROCK_COUNT`)
- Perlin noise-based clustering for natural distribution
- Avoids center area (bunker location)
- Correct heightmap sampling ensures rocks sit on terrain surface

**Technical Challenges Solved:**
- Rocks initially spawned before heightmap loaded (all at Y=-1.0)
- Fixed by checking `center_height > 0.0` before spawning
- Used separate `TerrainDecoration` marker to avoid terrain cleanup despawning rocks

**Sand Particles (Disabled):**
- Blowing sand particle system implemented but disabled
- Visual effect needs refinement for later iteration

### 10. Module Refactoring

**Directory:** [src/scenario/](src/scenario/)

Refactored scenario.rs (1177 lines) into organized submodules:
- `mod.rs` (510 lines) - Types, constants, plugin, initialization
- `wave.rs` (425 lines) - Wave state machine, spawning, reinforcements
- `ui.rs` (153 lines) - UI update systems
- `placement.rs` (134 lines) - Turret placement during preparation

### 11. Map Switching Cleanup

**File:** [terrain.rs](src/terrain.rs)

Fixed issues when switching between maps:

- Despawn all scenario units, towers, and shields on map change
- Clear squad manager state
- Reset game state
- Proper cleanup prevents entity accumulation across map switches

---

## Bug Fixes

1. **HP bar shrink direction** - Billboard rotation caused UV mirroring; fixed shader comparison
2. **Shield bar overlap** - Used camera-up vector offset instead of world-up
3. **Squad selection during prep** - Disabled selection system during preparation phase
4. **Hanabi particle warmup** - Fixed GPU shader compilation stutter on first explosion
5. **Scenario init ordering** - Ensured scenario spawns after map switch completes
6. **Rocks under terrain** - Changed Y offset from negative to positive

---

## Files Changed

| File | Lines | Description |
|------|-------|-------------|
| `src/scenario.rs` | +380 | Wave spawning, prep phase, turret placement |
| `src/terrain_decor.rs` | +491 | New: Rock decorations |
| `src/wfx_spawn.rs` | +467 | Turret explosion effects |
| `src/turrets.rs` | +270 | Health bars, death system, placement helpers |
| `src/objective.rs` | +230 | Tower/shield health bars |
| `src/terrain.rs` | +50 | Map cleanup improvements |
| `src/constants.rs` | +20 | Audio proximity, volume tuning |
| `src/particles.rs` | +60 | Turret explosion particles |
| `assets/shaders/health_bar.wgsl` | +26 | Health bar shader |
| `assets/shaders/shield_bar.wgsl` | +26 | Shield bar shader |

---

## Gameplay Flow

1. **Press 3** - Load Firebase Delta map
2. **Preparation Phase** - Place turrets (LMB), undo (RMB), toggle type (T), start (SPACE)
3. **Assault 1 Begins** - Enemies spawn from north in overlapping tactical waves
4. **Continuous Combat** - Waves blend together, no downtime between tactical waves
5. **Assault Cleared** - 30s cooldown, reinforcements arrive from south
6. **Repeat** - Assaults 2-3 with escalating enemy counts
7. **Victory** - Survive all 3 assaults (9 tactical waves total)
8. **Defeat** - Command bunker destroyed or breached by enemies

---

## Known Issues / Future Work

- Heat shimmer post-process effect (deferred to separate branch)
- Sand particle visual refinement
- Enemy AI for perimeter surrounding (simple, ~2-3 hours to implement)
- Final wave flanking from east spawn point (partially implemented)

---

## Debug Controls

| Key | Action |
|-----|--------|
| 1 | Flat terrain |
| 2 | Rolling Hills |
| 3 | Firebase Delta (scenario) |
| 7 | Test turret WFX explosion |
| 0 | Toggle debug mode |
| M | Toggle MG turret (debug) |
| H | Toggle Heavy turret (debug) |
| S | Destroy enemy shield (debug) |
