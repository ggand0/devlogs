# Devlog 027: Firebase Delta Scenario Implementation

**Date:** 2025-12-05

## Overview

Implemented the foundation for the Firebase Delta hilltop defense scenario - a wave-based defense mode where players defend a command bunker against 10 waves of enemies.

## Changes

### Phase 2: PNG Heightmap Loading

- Added `FirebaseDelta` variant to `MapPreset` enum
- Implemented async PNG heightmap loading with bilinear interpolation
- Added Key 3 handler for switching to Firebase Delta map
- Copied heightmap from `resources/heightmap/` to `assets/heightmap/`

### Phase 3: Scenario Architecture

**New file: `src/scenario.rs`** (~500 lines)
- Tunable constants: `TOTAL_WAVES`, `WAVE_SIZES`, `SPAWN_RATE`, etc.
- Resources: `ScenarioState`, `WaveManager`
- Components: `WaveEnemy`, `ScenarioUnit`, `CommandBunker`
- Wave state machine: Idle → Spawning → Fighting → Cooldown → Complete
- Scenario UI: Wave counter and enemy count display
- Victory/defeat check system

**Modified `src/setup.rs`:**
- Extracted `spawn_single_squad()` public helper
- Added `UnitMaterials` struct and `create_team_materials()` helper
- Added `create_droid_mesh()` public wrapper

**Modified `src/turrets.rs`:**
- Added `spawn_mg_turret_at()` and `spawn_heavy_turret_at()` public functions

### Phase 4-5: Partial

- Wave spawner system is placeholder (state machine works, actual spawning TBD)
- Basic UI framework in place

## Known Issues (To Fix)

1. Default 5,000 vs 5,000 units still spawn on FirebaseDelta map
2. Original two towers still spawn
3. Command bunker is buried under terrain

## Next Steps

- Disable default spawning for FirebaseDelta map
- Fix command bunker terrain height
- Implement actual enemy wave spawning
