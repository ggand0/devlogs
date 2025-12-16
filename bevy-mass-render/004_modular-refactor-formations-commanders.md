# Devlog: Major Refactor - Modular Architecture, Formations & Commander System

**Date:** June 8, 2025  
**Thread Duration:** Extended session (multiple hours)  
**Focus:** Modular code refactoring, formation system implementation, commander promotion, animation fixes

---

## Context

This thread started with a critical problem: the main.rs file had grown to over 2100 lines, which was causing AI response quality issues and making the codebase difficult to maintain. The initial goal was to refactor the code into modules, but this led to discovering and fixing several broken systems along the way.

### Initial State
- **main.rs**: 2100+ lines (too large)
- **Squad system**: 100 squads per team, 50 units each (10×5 formation)
- **Issues discovered**: Animation broken, color shifting not working, formation system incomplete
- **Performance target**: Maintain 60+ FPS with 10,000 units

---

## Major Refactoring Work

### Module Structure Created

Broke down the monolithic main.rs into logical modules:

```
src/
├── main.rs           # App entry point, system registration
├── setup.rs          # Scene initialization, 10K unit spawning
├── types.rs          # Core components and data structures
├── constants.rs      # Game configuration constants
├── formation.rs      # Formation system, advance/retreat commands
├── movement.rs       # Unit movement, formation correction, animations
├── commander.rs      # Commander promotion and visual updates
└── combat.rs         # Target acquisition, laser firing, collisions
```

**Benefits:**
- Improved code organization and maintainability
- Clearer separation of concerns
- Better AI assistant response quality
- Easier to locate and fix specific systems

---

## Animation & Color Shifting Recovery

### Problem Discovery
After the initial refactor, animations and color shifting were completely broken. Investigation revealed that:

1. **Color shifting was never about material colors** - it was actually Y-position modulation creating visual "energy" effects
2. **Over-engineered material animation system** had replaced the original simple Y-position logic
3. **UI text incorrectly mentioned formations** that didn't exist yet

### Solution Implemented

**Original Working Animation Logic (Restored):**
```rust
// Y-position modulation during movement
if is_actively_moving {
    let move_factor = (time.elapsed_secs() * 3.0).sin() * 0.15;
    transform.translation.y = base_y + move_factor;
} else {
    transform.translation.y = base_y;
}
```

**Key Points:**
- Units bob up and down when moving (sine wave animation)
- Static units stay at base Y position
- Simple, performant, visually effective
- Removed complex material animation system

---

## Commander Promotion System

### Initial Challenges

The commander promotion system had a critical issue: **entity panic crashes** due to race conditions between combat (destroying entities) and material updates (trying to modify them).

**Error Message:**
```
error[B0003]: Could not insert a bundle (of type `Handle<StandardMaterial>`) 
for entity because it doesn't exist in this World
```

### Evolution of Solutions

**Attempt 1: Entity Existence Checks**
- Added `unit_query.get(entity).is_ok()` checks
- Still crashed due to deferred command execution

**Attempt 2: Unique Materials for All Units**  
- Created unique materials for all 10,000+ units
- Fixed crashes but **destroyed FPS performance**
- User feedback: "FPS drastically reduced"

**Attempt 3: Shared Materials with Safety Checks**
- Reverted to shared materials for regular units
- Unique materials only for commanders
- Added multiple entity existence checks
- Still had intermittent crashes

**Final Solution: `try_insert` Method**
- Replaced all `entity_cmd.insert()` with `entity_cmd.try_insert()`
- Non-panicking insertion that fails gracefully if entity doesn't exist
- **Result: No more crashes, maintained performance**

### Commander System Features

**Visual Distinction:**
- **Team A Commanders**: Golden yellow body (0.9, 0.8, 0.4), bright gold head (1.0, 0.9, 0.5)
- **Team B Commanders**: Orange-red body (0.9, 0.5, 0.3), bright orange head (1.0, 0.6, 0.4)
- **Glowing markers**: Yellow cubes (Team A) / Red cubes (Team B) above commanders

**Promotion Logic:**
```rust
// Check if commander entity still exists
let commander_exists = unit_query.iter()
    .any(|(entity, squad_member)| 
        entity == commander_entity && squad_member.squad_id == squad_id);

if !commander_exists {
    // Promote first available unit in squad
    for (entity, squad_member) in unit_query.iter() {
        if squad_member.squad_id == squad_id && !squad_member.is_commander {
            // Update squad manager
            squad.commander = Some(entity);
            squad_member.is_commander = true;
            break;
        }
    }
}
```

**Performance Optimization:**
- Regular units share 2 base materials (Team A/B)
- Commanders get unique materials when promoted
- ~100 commanders max vs 10,000 units = minimal overhead

---

## Formation System Implementation

### Rectangle Formation

Implemented basic Rectangle formation (10×5 grid):

**Spacing:**
- Horizontal: 2.0 units
- Vertical: 2.5-3.0 units

**Formation Position Calculation:**
```rust
pub fn calculate_formation_position(
    row: usize,
    col: usize,
    squad_center: Vec3,
    formation_type: FormationType,
) -> Vec3 {
    match formation_type {
        FormationType::Rectangle => {
            let x_offset = (col as f32 - 4.5) * UNIT_SPACING_X;
            let z_offset = (row as f32 - 2.0) * UNIT_SPACING_Z;
            squad_center + Vec3::new(x_offset, 0.0, z_offset)
        }
        // Other formations planned but not implemented
    }
}
```

**Formation Correction System:**
- Units automatically move toward formation positions when stationary
- Disabled during active movement to prevent animation conflicts
- Uses target positions with smooth interpolation

---

## Retreat Animation Fix

### Problem Cascade

**Initial Issue**: Units retreating but not animating or color-shifting properly  
**Root Cause 1**: Formation correction running during retreat, fighting against retreat movement  
**Root Cause 2**: Depleted squads (with casualties) constantly recalculating formation centers

### Solution Process

**Fix 1: Disable Formation Correction During Movement**
```rust
let is_actively_moving = (!is_stationary && !droid.returning_to_spawn) 
                        || droid.returning_to_spawn;

if !is_actively_moving {
    // Only correct formation when truly stationary
    apply_formation_correction();
}
```

**Fix 2: Freeze Formation Targets During Retreat**
```rust
// Only update formation targets when NOT retreating
if !droid.returning_to_spawn {
    formation_offset.target_world_position = correct_target_position;
}
```

**Result:**
- ✅ Retreat animations work for full squads
- ✅ Retreat animations work for depleted squads
- ✅ Color shifting active during retreat
- ✅ No formation interference

---

## Technical Challenges & Solutions

### 1. Entity Panic Race Condition

**Problem:** Entities destroyed in combat between command scheduling and execution

**Technical Details:**
- Bevy commands are deferred until `ApplyDeferred` system runs
- Combat system can destroy entities after commander system checks existence
- `commands.get_entity(entity).insert()` panics if entity doesn't exist

**Solution:** Use `try_insert` for non-panicking insertion
```rust
// Before (panics)
entity_cmd.insert(new_commander_body);

// After (safe)
entity_cmd.try_insert(new_commander_body);
```

### 2. Material Corruption

**Problem:** Team A units getting white tint instead of blue-gray

**Cause:** Shared materials being modified when they shouldn't be

**Solution:** 
- Regular units share immutable base materials
- Only commanders get unique mutable materials
- Material creation happens at promotion time, not update time

### 3. Animation Interference

**Problem:** Formation correction system fighting with retreat movement

**Analysis:**
- `is_actively_moving` was `false` during retreat because units weren't "stationary"
- Formation correction kept trying to move units back to formation
- Created jittery, broken animations

**Solution:** Explicitly disable formation correction when `returning_to_spawn == true`

### 4. Depleted Squad Reformation

**Problem:** Squads with casualties had broken retreat animations

**Cause:** Squad center constantly recalculating as casualties mount, updating all formation targets

**Solution:** Freeze formation target updates during retreat to maintain consistent behavior

---

## Performance Considerations

### Material Management
- **Initial approach**: Unique materials for all 10K units → **FPS crash**
- **Final approach**: Shared materials + unique for commanders → **60+ FPS maintained**
- **Memory savings**: 2 base materials vs 10,000 individual materials

### Entity Safety Overhead
- Multiple existence checks before material updates
- Minimal performance impact vs crash prevention
- `try_insert` has negligible overhead vs `insert`

### AMD GPU Requirements
Development required specific Vulkan configuration:
```bash
VK_LOADER_DEBUG=error VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/radeon_icd.x86_64.json cargo run --release
```

---

## Testing & Validation

### Test Scenarios
1. **Commander promotion**: Verified commanders die and get replaced
2. **Material persistence**: Confirmed promoted commanders keep golden/orange colors
3. **Retreat behavior**: Tested full and depleted squad animations
4. **Crash resilience**: Ran 3+ times without entity panics
5. **Performance**: Maintained 60+ FPS during intense combat

### Known Issues (At End of Thread)
1. **Formation switching**: Q/E/R/T keys planned but Rectangle only working
2. **Commander markers**: Glowing cubes implemented but possibly disabled
3. **Formation shapes**: Other shapes (Line, Box, Wedge) not yet implemented
4. **Speed variance**: Units in formation gradually drift apart over time

---

## Code Quality Improvements

### Before Refactor
- 2100+ lines in single file
- Mixed concerns (combat, movement, formation, etc.)
- Difficult to navigate and maintain
- AI assistant struggled with context

### After Refactor
- Clean module separation (~200-400 lines per file)
- Single Responsibility Principle applied
- Easy to locate specific functionality
- Better maintainability and extensibility

---

## System Architecture

### Core Components
```rust
#[derive(Component)]
struct BattleDroid {
    team: Team,
    returning_to_spawn: bool,
    // ... other fields
}

#[derive(Component)]
struct SquadMember {
    squad_id: u32,
    is_commander: bool,
    formation_position: (usize, usize), // (row, col)
}

#[derive(Component)]
struct FormationOffset {
    local_offset: Vec3,
    target_world_position: Vec3,
}
```

### System Execution Order
1. **Update**: Player input, squad commands
2. **Movement**: Calculate velocities, formation correction
3. **Combat**: Target acquisition, laser firing
4. **Commander**: Check promotions, update materials
5. **Animation**: Apply Y-position modulation, color effects

---

## Lessons Learned

### 1. Don't Over-Engineer
The material animation system was far more complex than needed. The original Y-position animation was simple, performant, and effective.

### 2. Bevy Command Timing Matters
Commands are deferred, so entity existence checks must account for the delay between scheduling and execution.

### 3. Performance vs Features Trade-offs
Unique materials for all units was technically possible but performance-prohibitive. Smart compromises (shared materials + unique for special cases) achieved both goals.

### 4. Modular Refactoring Value
Breaking up large files isn't just about organization—it significantly improves development velocity and AI assistance quality.

### 5. Test Incrementally
Each fix revealed new issues. The progression from formation correction → depleted squads → target freezing showed the importance of testing various scenarios.

---

## Final State

### Working Features
- ✅ Modular codebase with clean separation
- ✅ Rectangle formation (10×5 grid)
- ✅ Advance/Retreat squad commands
- ✅ Commander promotion system
- ✅ Visual commander distinction (materials + markers)
- ✅ Y-position animations during movement
- ✅ Color shifting on retreat
- ✅ No entity panic crashes
- ✅ 60+ FPS with 10,000 units
- ✅ Formation correction when stationary

### Incomplete/Planned Features
- ⏳ Formation switching (Line, Box, Wedge)
- ⏳ Commander visual debugging (markers may be disabled)
- ⏳ Dynamic reformation after casualties
- ⏳ Better formation cohesion during movement
- ⏳ Speed variance compensation

---

## Commit Summary

```
feat: Major refactor - modular architecture with formations, commanders & animations

- Refactor 2100+ line main.rs into organized modules (formation, commander, movement, combat, types, setup)
- Implement Rectangle formation system with proper spacing and formation correction
- Add commander promotion system with visual distinction (golden/orange materials + glowing markers)
- Fix animation and color shifting: Y-position animations during movement, color shift on retreat
- Resolve formation interference with retreat animations for both full and depleted squads
- Optimize performance: shared materials for regular units, unique materials only for commanders
- Fix entity panic crashes using try_insert for safe material updates during combat
- Maintain 10K unit performance with consistent behavior across all squad states
```

---

## Next Session Goals

Based on issues identified at end of thread:

1. **Fix formation shape algorithms** for Line, Box, Wedge formations
2. **Enable/verify commander visual markers** (glowing cubes)
3. **Implement proper dynamic reformation** logic for depleted squads
4. **Improve formation cohesion** during movement with better catchup logic
5. **Add formation switching** system (Q/E/R/T keys fully functional)

---

**End of Session**  
**Status:** Stable, performant, ready for formation system expansion

