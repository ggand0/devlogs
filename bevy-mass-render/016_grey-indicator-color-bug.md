# Grey Move Indicator Color Bug

**Date:** 2025-11-29
**Issue:** Dead squad move indicators showing green instead of grey

## Problem Description

When a squad in a group dies, the move order system should display grey circle indicators at the dead squad's formation position (to show where it would have been). However, despite setting the material color to grey, the indicators were rendering as green - identical to living squad indicators.

### Symptoms
- Dead squad indicators were smaller (70% radius) - confirming the correct code path was executing
- Logs showed the grey color was being set correctly in the material
- But visually, all indicators appeared green

## Investigation

### Initial Debugging

Added extensive logging to track the color values:

```rust
info!("Creating material: is_custom={}, base_color={:?}, emissive={:?}",
      is_custom, base_color, emissive);
```

Logs confirmed:
```
Creating material: is_custom=true, base_color=Srgba(Srgba { red: 0.4, green: 0.4, blue: 0.4, alpha: 0.7 }), emissive=LinearRgba { red: 0.3, green: 0.3, blue: 0.3, alpha: 1.0 }
```

The material was being created with grey values, but rendering green.

### False Leads

1. **Color space conversion** - Suspected sRGB to Linear RGB conversion issues, but values were correct
2. **Material caching** - Suspected Bevy was reusing materials, but each `materials.add()` creates a new asset
3. **Mesh sharing** - Ruled out by making dead squad indicators smaller (0.7x radius)

### Root Cause Found

The bug was in `move_visual_cleanup_system` in [visuals.rs](../src/selection/visuals.rs):

```rust
// BUGGY CODE - hardcoded green color!
for (entity, mut visual, material_handle) in circle_query.iter_mut() {
    visual.timer.tick(time.delta());

    if let Some(material) = materials.get_mut(material_handle) {
        let progress = visual.timer.fraction();
        let alpha = (1.0 - progress) * 0.6;
        material.base_color = Color::srgba(0.2, 1.0, 0.3, alpha);  // <- ALWAYS GREEN!
    }
    // ...
}
```

This system runs every frame to fade out move indicators. It was **overwriting** the material's `base_color` to green regardless of what color the material was created with.

## Solution

### 1. Store Original Color in Component

Added `base_color` field to `MoveOrderVisual` component in [state.rs](../src/selection/state.rs):

```rust
#[derive(Component)]
pub struct MoveOrderVisual {
    pub timer: Timer,
    pub base_color: Color,  // Original color for fade-out
}
```

### 2. Set Color When Spawning

Updated `spawn_move_indicator_with_color` to store the base color:

```rust
commands.spawn((
    PbrBundle { /* ... */ },
    MoveOrderVisual {
        timer: Timer::from_seconds(MOVE_INDICATOR_LIFETIME, TimerMode::Once),
        base_color,  // Store original color for fade-out
    },
    // ...
));
```

### 3. Use Stored Color in Cleanup System

Fixed the cleanup system to preserve the original color:

```rust
for (entity, mut visual, material_handle) in circle_query.iter_mut() {
    visual.timer.tick(time.delta());

    if let Some(material) = materials.get_mut(material_handle) {
        let progress = visual.timer.fraction();
        let alpha = (1.0 - progress) * 0.6;
        // Use the stored base color, just update alpha
        material.base_color = visual.base_color.with_alpha(alpha);
    }
    // ...
}
```

## Files Changed

- `src/selection/state.rs` - Added `base_color` field to `MoveOrderVisual`
- `src/selection/visuals.rs` - Fixed `spawn_move_indicator_with_color` and `move_visual_cleanup_system`

## Key Takeaways

1. **Animation/cleanup systems can override material properties** - When debugging material colors not applying, check if any update systems are modifying the material after creation.

2. **Use component data to preserve state** - If a visual needs to maintain its original properties through animations, store those properties in the component.

3. **Size changes are a good debugging tool** - Changing the mesh size helped confirm the correct code path was executing, isolating the issue to the material/color.

## References

- [Change colour on a StandardMaterial - Bevy Discussion #6907](https://github.com/bevyengine/bevy/discussions/6907)
- [How to modify material property in bevy - Stack Overflow](https://stackoverflow.com/questions/75801912/how-to-modify-material-property-in-bevy)
- [Bevy 0.14 to 0.15 Migration Guide](https://bevy.org/learn/migration-guides/0-14-to-0-15/) - Notes on PbrBundle deprecation
