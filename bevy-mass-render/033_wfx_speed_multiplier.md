# Devlog 033: War FX Speed Multiplier & Sparkle Tuning

**Date**: 2025-12-12
**Branch**: `chore/debug-ui-fixes`
**Status**: Completed

## Overview

Added a global speed multiplier system to the War FX explosion effects (1.5x speed), then performed extensive tuning of sparkle emitters based on visual testing. Sparkles now have wider spread, slower velocity, longer lifetime, and constant gravity for a more dramatic, lingering effect.

---

## Problem

The War FX explosion effect, while visually accurate to the Unity reference, felt slightly sluggish in gameplay. The effect needed to be sped up uniformly across all 6 emitters without breaking the carefully tuned visual balance.

---

## Solution

### Architectural Approach

Instead of manually tweaking hundreds of individual timing values, implemented a **parametric speed multiplier** system that scales all timing-related parameters consistently.

### Key Design Principle

For a speed multiplier of `N`:
- **Time-based values** (lifetimes, delays, keyframe times) → **divide by N**
- **Rate-based values** (velocities, gravity, scroll speed) → **multiply by N**

This ensures physically correct acceleration while maintaining visual proportions.

---

## Implementation

### 1. Main Function Signature

Added `speed_mult: f32` parameter to `spawn_combined_explosion`:

```rust
pub fn spawn_combined_explosion(
    // ... existing parameters ...
    position: Vec3,
    scale: f32,
    speed_mult: f32,  // NEW: Global speed multiplier
)
```

### 2. Emitter Functions Updated

All 6 emitter spawn functions received the `speed_mult` parameter:
- `spawn_warfx_center_glow`
- `spawn_explosion_flames`
- `spawn_smoke_emitter`
- `spawn_glow_sparkles`
- `spawn_dot_sparkles`
- `spawn_dot_sparkles_vertical`

### 3. Timing Scaling Rules

#### Time Values (÷ speed_mult)
```rust
// Particle lifetimes
max_lifetime: 0.7 / speed_mult,  // Was 0.7s, now 0.467s at 1.5x

// Animation curve keyframe times
(0.0, value),           // Start
(0.10 / speed_mult, value),  // Was 10%, now 6.67%
(0.25 / speed_mult, value),  // Was 25%, now 16.67%
(1.0 / speed_mult, value),   // End (normalized by lifetime)

// Delays
spawn_delay: 0.5 / speed_mult,  // Was 0.5s, now 0.333s
```

#### Rate Values (× speed_mult)
```rust
// Velocities
velocity: 24.0 * scale * speed_mult,  // 1.5x faster movement

// Gravity
gravity: 4.0 * scale * speed_mult,    // 1.5x stronger pull

// UV scrolling
scroll_speed: Vec2::new(0.0, 10.0 * speed_mult),  // 1.5x faster scroll

// Rotation speed
rotation_speed: rng.gen_range(70.0..90.0) * speed_mult,  // 1.5x faster spin
```

### 4. Call Sites

Updated all spawn calls to use `1.5` as the speed multiplier:

**Debug hotkeys** (`src/objective.rs`):
```rust
// Keys 1-6 for individual emitters
spawn_combined_explosion(..., position, 4.0, 1.5);
```

**Tower explosions** (`src/explosion_system.rs`):
```rust
spawn_combined_explosion(..., position, 4.0, 1.5);
```

---

## Technical Details

### Animation Curve Keyframe Scaling

Animation curves use **normalized time** (0.0 to 1.0) within the particle's lifetime. The keyframe times must be scaled to compress the animation:

```rust
// Original (1.0x speed)
keyframes: vec![
    (0.0, 0.0),      // Start invisible
    (0.10, 1.0),     // Fade in at 10% of lifetime
    (0.25, 1.0),     // Hold until 25%
    (1.0, 0.0),      // Fade out at end
]

// Scaled (1.5x speed) - same visual progression, faster absolute time
keyframes: vec![
    (0.0, 0.0),              // Start invisible
    (0.10 / 1.5, 1.0),       // Fade in at 6.67% (arrives sooner)
    (0.25 / 1.5, 1.0),       // Hold until 16.67%
    (1.0 / 1.5, 0.0),        // Fade out at 66.67%
]
```

Since the lifetime itself is also divided by `speed_mult`, the **absolute wall-clock timing** works out correctly.

### UV Scroll Speed

The smoke shader uses a time-based UV offset that must scale with speed:

```rust
// In SmokeScrollMaterial
scroll_speed: Vec2::new(0.0, 10.0 * speed_mult),

// In shader (wfx_smoke_scroll.wgsl)
let scrolled_uv = uv + scroll_speed * time;  // Scrolls 1.5x faster
```

### Burst Delays

Multi-burst emitters (flames, sparkles) spawn particles in waves. Delays scale to maintain burst density:

```rust
// Original: 57 particles in 0.25-0.35s window
let delay = rng.gen_range(0.25..0.35);

// Scaled: Same 57 particles, but in 0.167-0.233s window
let delay = rng.gen_range(0.25..0.35) / speed_mult;
```

---

## Results

### Visual Impact (1.5x Speed)

| Component | Original | At 1.5x | Effect |
|-----------|----------|---------|--------|
| **Center glow** | 0.7s | 0.467s | Quicker flash |
| **Flame burst** | 3-4s | 2-2.67s | Faster dissipation |
| **Smoke delay** | 0.5s | 0.333s | Earlier onset |
| **Smoke lifetime** | 4.5-5.5s | 3-3.67s | Shorter lingering |
| **Sparkle velocity** | 24 u/s | 36 u/s | Faster spray |
| **Sparkle gravity** | 4 u/s² | 6 u/s² | Sharper arc |
| **UV scroll** | 10 u/s | 15 u/s | Faster turbulence |

### Total Effect Duration

- **Original**: ~5.5 seconds (longest particle lifetime)
- **At 1.5x**: ~3.67 seconds (5.5 / 1.5)
- **Reduction**: 33% faster completion

### Physics Accuracy

Despite the speed-up, all physics remain proportionally correct:
- Particle arc trajectories maintain the same **shape** (velocity and gravity both scale by 1.5)
- Smoke scrolling **rate** matches particle movement speed
- Fade timing **relative to lifetime** stays identical

---

## Files Modified

| File | Changes |
|------|---------|
| `src/wfx_spawn.rs` | Added `speed_mult` param to 7 functions, scaled ~120 timing values |
| `src/objective.rs` | Updated 7 debug hotkey calls with 1.5x multiplier |
| `src/explosion_system.rs` | Updated tower explosion call with 1.5x multiplier |

**Total changes**: 3 files, 119 insertions, 98 deletions

---

## Design Rationale

### Why 1.5x?

- **1.25x**: Too subtle, felt only marginally faster
- **1.5x**: Noticeable improvement, maintains visual quality
- **2.0x**: Too fast, loses cinematic feel and smoke detail

### Why Parametric vs. Manual Tuning?

**Parametric advantages**:
- ✅ Consistent scaling across all 6 emitters
- ✅ Easy to experiment (change one value, rebuild, test)
- ✅ Maintains relative timing relationships
- ✅ No risk of breaking carefully tuned curves

**Manual tuning risks**:
- ❌ 120+ values to adjust individually
- ❌ Easy to miss timing dependencies (e.g., delay vs. lifetime)
- ❌ Difficult to iterate (each change requires analyzing impact)
- ❌ No way to revert to "Unity-accurate" baseline

---

## Future Considerations

### Make Speed User-Configurable?

Could add a constant to allow easy adjustment:

```rust
// In constants.rs
pub const WFX_SPEED_MULTIPLIER: f32 = 1.5;

// In spawn calls
spawn_combined_explosion(..., WFX_SPEED_MULTIPLIER);
```

### Per-Emitter Speed Control?

Currently all emitters scale uniformly. Could expose individual control:

```rust
spawn_combined_explosion(..., EmitterSpeeds {
    glow: 1.5,
    flames: 1.5,
    smoke: 1.2,    // Keep smoke slower for lingering effect
    sparkles: 1.8, // Make sparkles faster for snappiness
});
```

This would allow fine-tuning the "feel" while maintaining the parametric approach.

---

## Testing

Tested via debug hotkeys:
- **Key 5**: Full combined explosion at 1.5x speed
- **Keys 1-6**: Individual emitters at 1.5x speed

Verified:
- ✅ All emitters complete ~33% faster
- ✅ No visual artifacts or broken curves
- ✅ Smoke scrolling matches particle movement
- ✅ Sparkle trajectories remain natural
- ✅ Burst timing maintains density

---

## Retrospective

### What Went Well

1. **Parametric design paid off** - Single parameter controls entire effect
2. **Clean implementation** - No magic numbers, all scaling is explicit
3. **Zero visual regressions** - Maintained quality while improving feel

### Lessons Learned

1. **Speed is subjective** - What feels "right" depends on gameplay context
2. **Uniform scaling works** - No need for per-emitter tuning (yet)
3. **Documentation matters** - Clear scaling rules (÷ for time, × for rate) prevented errors

---

## Post-Implementation: Sparkle Tuning (Same Session)

After implementing the speed multiplier, visual testing revealed sparkles felt too narrow and fast. Performed manual tuning:

### Sparkle Adjustments

**Size increase (40% larger)**:
- Glow sparkles: 0.15 → 0.21
- Dot sparkles: 0.2 → 0.28
- Dot sparkles vertical: 0.2 → 0.28

**Cone angle widening** (for horizontal spread):
- Glow sparkles: 70° → 90° (full hemisphere)
- Dot sparkles: 50° → 75°

**Velocity reduction** (~67% of original):
- Glow sparkles: 24 → 16
- Dot sparkles: 12-24 → 8-16
- Dot sparkles vertical: 6-12 → 6-8

**Lifetime increase** (2.5-4.4x original):
- Glow sparkles: 0.25-0.35s → 0.625-0.875s (2.5x)
- Dot sparkles: 0.2-0.3s → 0.875-1.3125s (~4.4x)
- Dot sparkles vertical: 0.1-0.3s → 0.4375-1.3125s (~3.3x)

**Gravity standardization**:
- All sparkle emitters now use constant `gravity: 9.8` m/s²
- Previously: scaled values (4.0, 2.0, 0.0) * scale * speed_mult
- Matches ground explosion spark behavior
- Creates nice realistic drag-down effect

### Result

Sparkles now have a more dramatic, lingering presence with realistic physics. They spray wider horizontally, move slower, and fall naturally under gravity, creating a better visual match to the overall explosion size and feel.

---

## Post-Implementation: Smoke Emitter Bug Fix (Follow-up Session)

During subsequent testing of the smoke emitter, discovered it had the same shader bug as the flame emitter from the previous session.

### Shader Bug Discovery

**Problem**: `wfx_smoke_only.wgsl` had hardcoded lifetime darkening calculation that completely overrode any ColorCurve changes made in Rust:

```wgsl
// BUGGY CODE (lines 40-47)
let lifetime_t = 1.0 - particle_alpha;
let lifetime_mult = mix(1.0, 0.3, lifetime_t);
let darkened_smoke = tint_color * lifetime_mult;
```

This was identical to the bug found in `wfx_smoke_scroll.wgsl` (flame emitter shader).

**Fix**: Removed hardcoded calculation, shader now uses `tint_color` directly from Rust:

```wgsl
// FIXED CODE
// NOTE: tint_color now contains the Color Over Lifetime result from Rust,
// so we apply it directly instead of calculating lifetime_mult here
let darkened_smoke = tint_color;
```

### Particle Spread Fix

**Problem**: Smoke particles appeared too condensed compared to Unity original and flame emitter.

**Original spawn pattern** (tight cone):
- Fixed radius: 1.6 units
- Tight cone: ~5 degree spread
- Limited vertical variation

**New spawn pattern** (full spherical distribution):
- Variable radius: 1.5-3.5 units (matches flame emitter)
- Full spherical distribution using theta (horizontal) and phi (vertical) angles
- Proper 3D spread across hemisphere

**Code change**:
```rust
// BEFORE
let radius = 1.6 * base_scale;
let spread = rng.gen_range(0.0..0.087); // ~5 degrees
let offset = Vec3::new(
    radius * spread * angle.cos(),
    vertical_spread * radius,
    radius * spread * angle.sin(),
);

// AFTER
let radius = rng.gen_range(1.5..3.5) * base_scale;
let theta = rng.gen_range(0.0..std::f32::consts::TAU);
let phi = rng.gen_range(0.3..std::f32::consts::PI);
let offset = Vec3::new(
    radius * phi.sin() * theta.cos(),
    radius * phi.cos(),
    radius * phi.sin() * theta.sin(),
);
```

### Color Over Lifetime

Confirmed Unity specification from `SMOKE_EMITTER_DETAILS.md`:
- Start Color: constant gray `rgb(0.725, 0.725, 0.725)`
- Color Over Lifetime: white (1.0) throughout - **no darkening**
- Alpha Over Lifetime: fade in (0→1 at 15%), hold (15-33%), fade out (33-100%)

Reverted to constant color curve matching Unity spec:
```rust
let color_curve = ColorCurve {
    keyframes: vec![
        (0.0, start_color),  // Gray throughout
        (1.0, start_color),
    ],
};
```

Experimental darkening curve commented out for potential future use.

### Testing Notes

- Smoke particles now spread naturally across large volume
- Constant gray color matches Unity reference
- Shader bug fix allows future color curve experimentation if needed
- On sandy/bright backgrounds, smoke appears brighter due to multiply blend (expected behavior)
- On black backgrounds, smoke appears as neutral gray (correct)

---

## Reference

- Original WFX implementation: Devlog 013
- Speed multiplier inspired by Unreal Engine's "Rate Scale" parameter on particle emitters
- Gravity approach inspired by UE5 ground explosion sparks
- Shader debugging pattern established: always check if shader has hardcoded logic that overrides uniform values
- Unity Asset: [War FX](https://assetstore.unity.com/packages/vfx/particles/war-fx-5669)
