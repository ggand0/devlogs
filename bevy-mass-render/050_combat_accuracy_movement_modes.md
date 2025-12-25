# 050: Combat Accuracy & Movement Modes

## Overview

Implemented two related tactical features:
1. **Movement Modes** - EaW-style Move/AttackMove/Hold system with H key toggle
2. **Hit Probability** - Replaced deterministic 100% hit rate with probability-based accuracy featuring tactical modifiers

## Feature 1: Movement Modes

### MovementMode Component

```rust
#[derive(Component, Default, Clone, Copy, PartialEq, Eq, Debug)]
pub enum MovementMode {
    #[default]
    Move,        // March to destination, fire on the move (default)
    AttackMove,  // Stop when engaged with an enemy
    Hold,        // Stay in position, don't move at all
}
```

### Behavior

| Mode | Movement | Firing | Use Case |
|------|----------|--------|----------|
| Move | Always march to target | Fire on the move | Default, aggressive advance |
| AttackMove | Stop when has target | Fire when stopped | Cautious advance |
| Hold | No movement | Fire in place | Defense, ambush |

### Keybinds

| Key | Action |
|-----|--------|
| `H` | Toggle Hold mode for selected units |
| `Shift+RMB` | Attack Move command (units stop when engaged) |
| `RMB` | Move command (default, fire on the move) |

### Implementation

**animate_march() (movement.rs)**
```rust
let mode_allows_movement = match movement_mode {
    MovementMode::Hold => false,
    MovementMode::AttackMove => combat_unit.current_target.is_none(),
    MovementMode::Move => true,
};

let should_move = if !mode_allows_movement {
    false
} else if droid.returning_to_spawn {
    // Check distance to spawn...
} else {
    // Check distance to target...
};
```

**hold_command_system (selection/input.rs)**
- H key toggles Hold/Move for selected squads
- Smart toggle: if majority holding, switch all to Move; otherwise switch all to Hold
- Logs unit count and new mode

### Query Changes

Modified `animate_march` query to include new components:
```rust
Query<(Entity, &mut BattleDroid, &mut Transform, &SquadMember, &MovementMode, &CombatUnit), ...>
```

## Feature 2: Hit Probability System

### Accuracy Constants (constants.rs)

```rust
// Base accuracy
pub const INFANTRY_BASE_ACCURACY: f32 = 0.70;    // 70% for droids
pub const TURRET_BASE_ACCURACY: f32 = 0.80;      // 80% for turrets

// Modifiers
pub const ACCURACY_STATIONARY_BONUS: f32 = 0.15;        // +15% if shooter stationary
pub const ACCURACY_HIGH_GROUND_BONUS: f32 = 0.10;       // +10% if shooter higher
pub const HIGH_GROUND_HEIGHT_THRESHOLD: f32 = 3.0;      // Need 3+ units height advantage
pub const ACCURACY_TARGET_MOVING_PENALTY: f32 = 0.10;   // -10% if target moving

// Range falloff
pub const ACCURACY_RANGE_FALLOFF_START: f32 = 50.0;     // No penalty under 50 units
pub const ACCURACY_RANGE_FALLOFF_PER_50U: f32 = 0.05;   // -5% per 50 units beyond

// Clamps
pub const ACCURACY_MIN: f32 = 0.30;  // Always at least 30% hit chance
pub const ACCURACY_MAX: f32 = 0.95;  // Always some miss chance
```

### MovementTracker Component

Tracks whether units are stationary for accuracy bonuses:

```rust
#[derive(Component)]
pub struct MovementTracker {
    pub last_position: Vec3,
    pub stationary_timer: f32,  // Time spent not moving
    pub is_stationary: bool,    // True if stationary > 0.5s
}
```

**update_movement_tracker system (movement.rs)**
```rust
pub fn update_movement_tracker(
    time: Res<Time>,
    mut query: Query<(&Transform, &mut MovementTracker), With<BattleDroid>>,
) {
    for (transform, mut tracker) in query.iter_mut() {
        let distance_moved = (current_pos - tracker.last_position).length();

        if distance_moved < ACCURACY_MOVEMENT_THRESHOLD {
            tracker.stationary_timer += delta_time;
            tracker.is_stationary = tracker.stationary_timer >= ACCURACY_STATIONARY_TIME_THRESHOLD;
        } else {
            tracker.stationary_timer = 0.0;
            tracker.is_stationary = false;
        }
        tracker.last_position = current_pos;
    }
}
```

### calculate_hit_chance Function (combat.rs)

```rust
pub fn calculate_hit_chance(
    base_accuracy: f32,
    shooter_pos: Vec3,
    target_pos: Vec3,
    shooter_stationary: bool,
    target_stationary: bool,
) -> f32 {
    let mut accuracy = base_accuracy;

    // Stationary shooter bonus (infantry only - turrets always stationary)
    if shooter_stationary && base_accuracy < 0.75 {
        accuracy += ACCURACY_STATIONARY_BONUS;
    }

    // High ground bonus
    if shooter_pos.y > target_pos.y + HIGH_GROUND_HEIGHT_THRESHOLD {
        accuracy += ACCURACY_HIGH_GROUND_BONUS;
    }

    // Target moving penalty
    if !target_stationary {
        accuracy -= ACCURACY_TARGET_MOVING_PENALTY;
    }

    // Range falloff
    let distance = shooter_pos.distance(target_pos);
    if distance > ACCURACY_RANGE_FALLOFF_START {
        let range_penalty = ((distance - ACCURACY_RANGE_FALLOFF_START) / 50.0)
            * ACCURACY_RANGE_FALLOFF_PER_50U;
        accuracy -= range_penalty;
    }

    accuracy.clamp(ACCURACY_MIN, ACCURACY_MAX)
}
```

### Integration Points

**Infantry Hitscan (hitscan_fire_system)**
```rust
// Roll for hit before applying damage
let hit_chance = calculate_hit_chance(
    INFANTRY_BASE_ACCURACY,
    shooter_pos,
    target_pos,
    shooter_tracker.is_stationary,
    target_tracker.is_stationary,
);

if rng.gen::<f32>() > hit_chance {
    continue; // Miss - still spawn tracer for visual
}
// Hit - mark target for death
```

**Turret Projectiles (collision_detection_system)**
```rust
// When projectile hits, roll for accuracy
let hit_chance = calculate_hit_chance(
    TURRET_BASE_ACCURACY,
    projectile.origin,  // Stored where shot was fired
    droid_transform.translation,
    true,  // Turrets always stationary
    movement_tracker.is_stationary,
);

if rng.gen::<f32>() > hit_chance {
    continue; // "Grazes" but no damage
}
// Hit - mark for death
```

### LaserProjectile Origin Field

Added to track shooter position for turret accuracy calculations:
```rust
pub struct LaserProjectile {
    pub velocity: Vec3,
    pub lifetime: f32,
    pub team: Team,
    pub origin: Vec3,  // NEW: Where the shot originated
}
```

## Accuracy Examples

| Scenario | Calculation | Final |
|----------|-------------|-------|
| Infantry, both moving, 50u | 70% - 10% (target) | 60% |
| Infantry, stationary vs moving, 50u | 70% + 15% - 10% | 75% |
| Infantry, stationary vs stationary, high ground, 50u | 70% + 15% + 10% | 95% (cap) |
| Infantry, both moving, 150u | 70% - 10% - 10% (range) | 50% |
| Turret vs moving target, 100u | 80% - 10% - 5% | 65% |
| Turret vs stationary, close | 80% | 80% |

## Files Modified

| File | Changes |
|------|---------|
| `src/types.rs` | Added `MovementMode`, `MovementTracker` components, `origin` to LaserProjectile |
| `src/constants.rs` | Added accuracy constants |
| `src/setup.rs` | Spawn new components with droids |
| `src/movement.rs` | Mode check in animate_march, tracker update system |
| `src/combat.rs` | `calculate_hit_chance()`, modified hitscan + projectile systems |
| `src/selection/input.rs` | H key hold_command_system |
| `src/selection/movement.rs` | Shift+RMB Attack Move, sets movement mode on move command |
| `src/selection/mod.rs` | Exported hold_command_system |
| `src/main.rs` | Registered new systems |

## Future Work

- **Stuck Prevention** - Clear target after 2s without LOS to prevent deadlock
- **Suppression** - Units under fire have reduced accuracy
- **Cover System** - Terrain objects provide accuracy bonus
- **Visual Feedback** - Miss tracers slightly offset from target
- **Mode UI Indicator** - Visual indicator showing current movement mode
