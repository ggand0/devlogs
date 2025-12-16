# Devlog 034: War FX Debug Spawn Fix for Firebase Delta

**Date**: 2025-12-15
**Branch**: `chore/debug-ui-fixes`
**Status**: Completed

## Overview

Fixed War FX debug explosions being buried underground on Firebase Delta (map 3) by spawning them at the same offset position used by the UE ground explosion debug menu.

---

## Problem

When testing War FX explosions on Firebase Delta using debug keys (0 to toggle, 1-7 for emitters), the effects spawned at position `(0, 10, 0)` which is below the terrain surface on this map. The explosions were invisible because they rendered underground.

---

## Solution

Updated `debug_warfx_test_system` in `src/objective.rs` to:

1. Add `terrain_config` and `heightmap` parameters to query current map and terrain height
2. For Firebase Delta: spawn at offset position `(40, terrain_y, 30)` matching UE ground explosion debug menu
3. For other maps: keep original spawn at `(0, 10, 0)`

---

## Implementation

```rust
pub fn debug_warfx_test_system(
    // ... existing parameters ...
    terrain_config: Res<crate::terrain::TerrainConfig>,
    heightmap: Res<crate::terrain::TerrainHeightmap>,
) {
    // For each debug key (1-7):
    let position = if terrain_config.current_map == crate::terrain::MapPreset::FirebaseDelta {
        let offset_x = 40.0;
        let offset_z = 30.0;
        let terrain_y = heightmap.sample_height(offset_x, offset_z);
        Vec3::new(offset_x, terrain_y, offset_z)
    } else {
        Vec3::new(0.0, 10.0, 0.0)
    };
    // ... spawn emitter at position ...
}
```

All 7 debug keys now use this map-aware positioning.

---

## Files Modified

| File | Changes |
|------|---------|
| `src/objective.rs` | Added terrain params, updated spawn positions for keys 1-7 |

---

## Testing

- Verified on Firebase Delta: explosions spawn above terrain at offset position
- Verified on other maps: explosions spawn at original `(0, 10, 0)` position
- All 6 emitters render correctly on both map types

---

## Summary: WFX Tuning on This Branch

This branch (`chore/debug-ui-fixes`) included several War FX explosion improvements:

### 1. Speed Multiplier (1.5x)

Added a parametric `speed_mult` parameter to all 6 emitter spawn functions. Time-based values (lifetimes, delays, keyframe times) are divided by the multiplier, while rate-based values (velocities, gravity, scroll speed) are multiplied. This ensures physically correct scaling - particle arcs maintain the same shape while completing 33% faster.

### 2. Sparkle Emitter Tuning

Adjusted all three sparkle emitters (glow sparkles, dot sparkles, dot sparkles vertical) based on visual testing:
- **Size**: Increased 40% (0.15 → 0.21 for glow, 0.2 → 0.28 for dots)
- **Cone angle**: Widened horizontal spread (70° → 90° for glow, 50° → 75° for dots)
- **Velocity**: Reduced to ~67% (24 → 16 for glow, 12-24 → 8-16 for dots)
- **Lifetime**: Increased 2.5-4.4x for longer lingering effect
- **Gravity**: Standardized to constant 9.8 m/s² (was scaled values) for realistic fall-off

### 3. Shader Bug Fix (ColorCurve)

Both `wfx_smoke_scroll.wgsl` (flame emitter) and `wfx_smoke_only.wgsl` (smoke emitter) had hardcoded lifetime darkening that completely ignored the ColorCurve values sent from Rust:

```wgsl
// BUG: This overrode any Rust-side color changes
let lifetime_mult = mix(1.0, 0.3, 1.0 - particle_alpha);
let darkened_smoke = tint_color * lifetime_mult;
```

Fixed by removing the hardcoded calculation and using `tint_color` directly, which now contains the proper Color Over Lifetime result from the Rust ColorCurve system.

### 4. Smoke Emitter Particle Spread

Smoke particles were too condensed compared to the flame emitter and Unity reference. Changed spawn distribution from a tight ~5° cone to full spherical distribution:

```rust
// BEFORE: Tight cone
let radius = 1.6 * base_scale;
let spread = rng.gen_range(0.0..0.087); // ~5 degrees

// AFTER: Full spherical (matches flame emitter)
let radius = rng.gen_range(1.5..3.5) * base_scale;
let theta = rng.gen_range(0.0..TAU);           // 0-360° horizontal
let phi = rng.gen_range(0.3..PI);              // hemisphere vertical
```

Also reverted color curve to constant gray `rgb(0.725, 0.725, 0.725)` per Unity spec - the original has no color darkening over lifetime, only alpha fade.

### 5. Firebase Delta Debug Spawn Fix

Debug explosions (keys 1-7) now spawn at terrain-aware positions on Firebase Delta instead of being buried underground at `(0, 10, 0)`.
