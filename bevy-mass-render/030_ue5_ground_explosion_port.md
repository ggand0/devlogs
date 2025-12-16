# Devlog 030: UE5 Ground Explosion Port

## Overview

Ported UE5's `NS_Explosion_Sand_5` Niagara particle system to Bevy as a new ground explosion effect. This creates large-scale artillery/explosive impacts with multiple layers of effects.

## Implementation

### New Module: `src/ground_explosion.rs`

Created a complete ground explosion system with:

- **FlipbookMaterial**: Custom material for sprite sheet animation with UV offset calculation
- **FlipbookSprite component**: Tracks animation state (frame, elapsed time, lifetime)
- **VelocityAligned component**: Orients billboards along their velocity vector
- **BottomPivot component**: Marks sprites that grow upward from their base (fireballs)
- **CameraFacing component**: Standard billboarding toward camera
- **SpriteRotation component**: Random rotation around billboard's facing axis
- **DirtPhysics component**: Velocity + gravity + drag for dirt particles
- **Dirt001ScaleOverLife component**: Linear growth curve for velocity-aligned dirt

### Emitters Implemented

| Key | Emitter | Description | Status |
|-----|---------|-------------|--------|
| 1 | Main Fireball | 8x8 flipbook, velocity-aligned, bottom pivot, HSV color variation | ✅ Complete |
| 2 | Secondary Fireball | 8x8 flipbook, velocity-aligned, bottom pivot, HSV color variation | ✅ Complete |
| 3 | Dirt | Billboard chunks with gravity, scale shrink, XY flattening | ✅ Complete |
| 4 | Dirt001 | Velocity-aligned debris, no gravity, scale growth | ✅ Complete |
| 5 | Dust Ring | 4x1 flipbook, 35° cone, velocity-aligned, 3× brightness | ✅ Complete |
| 6 | Wisp | 8x8 flipbook, gravity arc motion, 1.5× scale | ✅ Complete |
| 7 | Smoke Cloud | 8x8 flipbook, camera-facing, color/alpha over life | ✅ Complete |
| 8 | Sparks | Both spark + flash spark emitters | ✅ Complete |
| 9 | Parts | 3D mesh debris, gravity, bounce, tumbling rotation | ✅ Complete |
| J | Group 1-6 | main, main001, dirt, dirt001, dust, wisp | ✅ Working |
| K | Full Explosion | All emitters combined | ✅ Working |
| - | Impact Flash | Tilted ground flash, 70° angle | ⚠️ Needs work |

### UE5-Accurate Features Implemented

#### Fireball (main/main001)
- **Spawn delay**: 0.05s (negative elapsed time)
- **Spawn position**: 50-unit sphere offset
- **Cone velocity**: 90° hemisphere, 450-650 units speed
- **HSV color variation**: Hue ±10%, Saturation 80-100%, Value 80-100%
- **Sprite rotation**: Random 0-360°
- **Scale curve**: 0.5→2.0 over lifetime (cubic ease-out)
- **Alpha curve**: Hold at 1.0 until t=0.8, then cubic fade to 0

#### Smoke
- **Camera-facing**: Unaligned mode with sprite rotation
- **Color over life**: RGB 0.4→0.2, Alpha 0.6→0
- **Scale over life**: 1→3x with ease-out
- **Camera-local velocity**: Moves relative to camera view

#### Dirt (billboard debris)
- **Spawn delay**: 0.1s
- **Count**: 35 fixed
- **Box velocity**: XY ±500, Z 1000-1500 (strong upward launch)
- **Physics**: Gravity 9.8 m/s², Drag 2.0
- **Color**: Dark brown (0.082, 0.063, 0.050) → lighter brown
- **Scale curve**: Linear shrink 100→0%
- **Size XY curve**: X stretches (1→2), Y compresses (1→0.5) - "flattening" effect
- **Alpha curve**: Fast fade-in (0→1.0 in first 10%), slow cubic fade-out (90%)
- **Sprite rotation**: Random 0-360°

#### Dirt001 (velocity-aligned debris)
- **Spawn delay**: 0.1s
- **Count**: 10-15 random
- **Cone velocity**: 90° hemisphere, 250-1000 units speed, falloff toward center
- **No gravity**: Debris shoots outward and fades without falling
- **Color**: Same dark brown as dirt
- **Scale curve**: Linear GROWTH 1→2x (opposite of dirt!)
- **Alpha curve**: Same as dirt (fast fade-in, slow fade-out)
- **Sprite rotation**: Random 0-360°

#### Wisp (smoke tendrils)
- **Count**: 3 particles (single set, simplified from original double-spawn)
- **Scale**: 1.5× base scale for visibility
- **Size**: 0.8-1.8m base × 1.5× scale × 5× scale curve = ~6-13.5m final
- **Arc motion**: Gravity-based (9.8 m/s²) - gentle upward launch (3-6 m/s), then falls
- **Color**: Dark grey-black (0.15, 0.12, 0.10) - matches dust
- **Alpha curve**: 4.0→1.0 in first 20%, then 1.0→0 linear (UE5 brightness multiplier)
- **Scale curve**: 0→5× cubic ease-in
- **Lifetime**: 1.0-2.0s (reduced from 2-9s for punchier effect)

#### Dust Ring (upward dust column)
- **Count**: 2 sets of 2-3 particles (4-6 total), second set with 0.05s delay
- **Size**: 6-10m base with (3×, 2×) XY scale curve
- **Velocity**: 35° narrow cone, 500-1000 units speed
- **Color**: Dark grey-black (0.15, 0.12, 0.10)
- **Alpha curve**: 3.0→0 linear fade (UE5 brightness multiplier)
- **Lifetime**: Very short (0.1-0.5s)

#### Sparks (flying embers)
- **Count**: 30-60 (UE5: 250-500, reduced for performance)
- **Velocity**: 90° hemisphere cone, 15-37.5m speed with center falloff (1.5× UE5 for higher launch)
- **Physics**: Gravity 6.0 m/s² (UE5: 9.8, reduced for higher arc to match SFX)
- **HDR Color Curve**: (50/27/7.6) → (1/0.05/0) over 55% lifetime - "cooling ember" effect
- **Flickering**: sin(time + random_phase) per particle
- **Size**: 0.8-1.8m (UE5: 1-3 units, scaled up for visibility with blast SFX)
- **Spawn Height**: Y + 3.0×scale (raised to match fireball center)

#### Flash Sparks (spark_l - ring burst)
- **Count**: 20-50 (UE5: 100-250, reduced for performance)
- **Ring Spawn**: Particles spawn at sphere equator (horizontal ring)
- **Velocity**: 100° cone (wider than hemisphere), 4-55m speed
- **Physics**: Deceleration (-0.25, -1.0, -0.5) instead of gravity
- **Constant HDR**: (10/6.5/3.9) orange with linear alpha fade
- **"Shooting Star" XY Scale**: t=0: 0×0 → t=0.05: 0.3×50 → t=0.5: 5×3 (elongated then normalizes)
- **Flickering**: sin(time + random_phase) per particle
- **Size**: 0.4-1.0m (UE5: 0.05-1.0 units, scaled up for visibility)
- **Spawn Height**: Y + 2.5×scale (raised to match fireball)

#### Parts (3D mesh debris)
- **Count**: 50-75 (UE5: 50-75, matching spec)
- **Mesh**: 3 variants (cube, flat slab, elongated piece) using Bevy Cuboid primitives
- **Size**: 0.3-0.5m (UE5: 5-7 units = 0.05-0.07m, scaled ~6× for subtle visibility)
- **Velocity**: Box distribution ±8m/s horizontal, 5-25m/s upward
- **Physics**: Gravity 9.8 m/s², 1 bounce with friction 0.25
- **Rotation**: Random initial rotation + tumbling angular velocity (±10 rad/s per axis)
- **Color**: Mid brown-grey (0.35, 0.30, 0.24)
- **Lifetime**: 0.5-1.5s
- **Scale Curve**: Quick grow (0-10%), hold at 1.0 (10-90%), fast shrink (90-100%)

### Size Scaling (Bevy vs UE5)

All sizes documented with UE5 spec values. Since smaller emitters were scaled up for visibility,
the fireball was scaled down to maintain balanced proportions.

| Emitter | UE5 Spec | Current Bevy | Previous Bevy | Notes |
|---------|----------|--------------|---------------|-------|
| Fireball | 2500-2600 units (25-26m base) | 14-18m base | 20-26m base | Reduced to balance with scaled-up emitters |
| Dirt | 50-100 units (0.5-1.0m) | 1-2m | 8-14m | 2× UE5 spec (reduced from 8-14m) |
| Dirt001 | 50-100 units (0.5-1.0m) | 1-2m | 8-14m | 2× UE5 spec (reduced from 8-14m) |
| Dust Ring | 300-500 units (3-5m) | 6-10m base | - | 2× scale, double-spawn (2 sets) |
| Wisp | 80-180 units (0.8-1.8m) | 1.2-2.7m base | 2.4-5.4m | 1.5× scale, single set, gravity arc |
| Sparks | 1-3 units (0.01-0.03m) | 0.8-1.8m | 0.5-1.2m | Increased for blast SFX match |
| Flash Sparks | 0.05-1.0 units | 0.4-1.0m | 0.15-0.4m | Increased 2.5× for fireball match |
| Parts | 5-7 units (0.05-0.07m) | 0.3-0.5m | - | 3D mesh, small for subtle despawn |

### Spawn Height Adjustments

Spark emitters spawn height increased to match fireball vertical extent:

| Emitter | Current Height | Previous Height | Notes |
|---------|----------------|-----------------|-------|
| Sparks | Y + 3.0×scale | Y + 0.5×scale | 6× increase to match 14-18m fireball |
| Flash Sparks | Y + 2.5×scale | Y + 0.5×scale | 5× increase to match fireball |

### Double-Spawn Pattern

Used for dust emitter to improve visibility without increasing individual particle size:

```
for set in 0..2 {
    let spawn_delay = if set == 0 { 0.0 } else { -0.05 };  // 50ms delay for second set
    for i in 0..count_per_set {
        // spawn particle with elapsed: spawn_delay
    }
}
```

| Emitter | Particles per Set | Total | Delay |
|---------|-------------------|-------|-------|
| Dust Ring | 2-3 | 4-6 | 0.05s |

Note: Wisp was simplified to a single set of 3 particles with gravity-based arc motion for a punchier effect.

### Debug Menu

Press **P** to toggle on-screen debug menu (bottom-left), then:
- **1-9**: Test individual emitters (see table above for mapping)
- **J**: Spawn emitter group 1-6 (main, main001, dirt, dirt001, dust, wisp)
- **K**: Spawn full ground explosion (all emitters)
- **P**: Close menu

Map switching moved to **F1-F4** (Flat, RollingHills, FirebaseDelta, Debug).

### Technical Details

**Flipbook UV Calculation** (in `flipbook.wgsl`):
```wgsl
let frame_col = frame_data.x;
let frame_row = frame_data.y;
let columns = frame_data.z;
let rows = frame_data.w;

let frame_size_x = 1.0 / columns;
let frame_size_y = 1.0 / rows;

let frame_offset = vec2<f32>(frame_col * frame_size_x, frame_row * frame_size_y);
let frame_uv = in.uv * vec2<f32>(frame_size_x, frame_size_y) + frame_offset;
```

**Velocity-Aligned Billboard**:
- Sprite Y-axis points along velocity direction
- Falls back to camera-facing when velocity is near zero
- Applies SpriteRotation around velocity axis

**Spawn Delay System**:
- Uses negative `elapsed` time to delay particle spawn
- Particle starts `Visibility::Hidden`
- Becomes visible when `elapsed >= 0`

**Alpha Curves**:
- Fireball: Hold at 1.0 until t=0.8, then quadratic fade
- Dirt/Dirt001: Fast fade-in (10%), slow cubic fade-out (90%)
- Smoke: Linear fade with color curve

**UE5 Brightness Multiplier** (in `flipbook.wgsl`):
UE5 uses alpha values > 1.0 as brightness multipliers (e.g., 3.0 = 3× brighter).
This is how dust/wisp emitters achieve their initial bright appearance.
```wgsl
let alpha_multiplier = color_data.a;
let tinted_color = sprite_sample.rgb * color_data.rgb * alpha_multiplier;
let final_alpha = sprite_sample.a * min(alpha_multiplier, 1.0);
```

## Issues & Fixes Applied

### F3 Smoke - "Dancing Tornadoes"
- **Problem**: Originally had velocity-aligned smoke, looked unnatural
- **Fix**: Changed to stationary camera-facing billboards with camera-local velocity

### F4 Wisp - Double Animation
- **Problem**: 3 particles with lifetime variation caused animation to appear twice
- **Fix**: Single large billboard (6-10m), plays 64 frames once over 1 second

### F8 Impact - White Square / 16-bit Style Glow
- **Problem**: AdditiveMaterial uses luminance-based alpha calculation
- **Fix**: Using `WFX_T_GlowCircle A8.png` designed for luminance-based additive blending

### F9 Dirt - Wrong Texture
- **Problem**: Was using T_Dirt_1.PNG instead of T_Dirt_3.PNG
- **Fix**: Replaced dirt.png with correct T_Dirt_3.PNG from UE5 project

### Dirt/Dirt001 - Too Small
- **Problem**: UE5 spec sizes (0.5-1.0m) were barely visible
- **Fix**: Scaled up to 2.0-3.5m with documented UE5 original values

## Assets

Textures stored in `assets/textures/premium/ground_explosion/`:
- `main_8x8.png` - Primary fireball (8x8 grid, 64 frames)
- `secondary_8x8.png` - Secondary fireball (8x8 grid)
- `smoke_8x8.png` - Smoke cloud (8x8 grid)
- `wisp_8x8.png` - Wisp smoke (8x8 grid)
- `dust_4x1.png` - Dust ring (4x1 strip)
- `dirt.png` - Dirt debris (T_Dirt_3.PNG)
- `flare.png` - Bright flash
- `impact.png` - Ground impact

## Reference Documentation

UE5 emitter specs extracted to `resources/ue5_ground_explosions/emitters/`:
- `main.md` - Primary fireball parameters
- `main001.md` - Secondary fireball parameters
- `dirt.md` - Billboard dirt debris parameters
- `dirt001.md` - Velocity-aligned dirt parameters
- `spark.md` - Spark emitter parameters

### SFX Integration

Sound effects added with random selection:
- `ground_explosion0.wav` - Blast sound variant 1
- `ground_explosion1.wav` - Blast sound variant 2

Played automatically when spawning full explosion (K key in debug menu).

### Timing Adjustments for SFX

Fireball lifetime extended to match lingering SFX:

| Emitter | UE5 Lifetime | Current Lifetime | Notes |
|---------|--------------|------------------|-------|
| Main Fireball | 1.0s | 1.5s | Extended for SFX linger |
| Secondary Fireball | 1.0s | 1.5s | Extended for SFX linger |

Spark physics tuned for higher arc to match blast SFX:

| Parameter | UE5 Value | Current Value | Notes |
|-----------|-----------|---------------|-------|
| Speed | 10-25 m/s | 15-37.5 m/s | 1.5× for higher launch |
| Gravity | 9.8 m/s² | 6.0 m/s² | Reduced for higher arc |

### Recent Tuning Changes

#### Fireball Scale & Velocity
Reduced fireball to avoid vertical stretching:
- **Scale curve**: 0.5→1.3 (was 0.5→2.0)
- **Velocity**: 3-5 m/s (was 4.5-6.5 m/s)

#### Dirt Upward Velocity
Increased for higher arc:
- **Upward velocity**: 15-25 m/s (was 10-15 m/s)

#### Spark Spawn Position
Reset to explosion core (removed Y offset that was added earlier).

#### Wisp Rework
Simplified for punchier, dirt001-like motion:
- Disabled double-spawn (was 2 sets of 3, now single set of 3)
- Scale reduced to 1.5× (was 2×)
- Added gravity (9.8 m/s²) for arc motion like dirt001
- Velocity reduced to 3-6 m/s upward (was complex two-phase)
- Alpha curve changed: 4.0→1.0 at t=0.2, then fade (was 3.0→1.0 at t=0.1)
- Lifetime reduced to 1.0-2.0s (was 2-9s)

#### Dirt Size Reduction
Reduced dirt emitter sizes from 8-14m to 1-2m (2× UE5 spec of 0.5-1.0m):
- **Dirt (billboard)**: 1-2m (was 8-14m)
- **Dirt001 (velocity-aligned)**: 1-2m (was 8-14m)

#### Fireball Alpha Curve
Changed from "hold-then-fade" to UE5-accurate S-curve fade:
- **Old**: Hold at 1.0 until t=0.8, then quadratic fade
- **New**: Smoothstep fade throughout entire lifetime
- LUT: t=0→1.0, t=0.2→0.77, t=0.4→0.56, t=0.6→0.33, t=0.8→0.14, t=1.0→0.0
- Added `FireballAlphaCurve` component marker

#### UV Zoom (Disabled)
UE5 spec indicates UV scale 500→1 over lifetime, but this doesn't translate directly:
- Added `uv_scale` field to `FlipbookMaterial` and shader support
- Added `FireballUVZoom` component and `update_fireball_uv_zoom` system
- **Currently disabled** - UE5's "500→1" UV scale likely works differently than simple UV division
- May need investigation of UE5's material shader or different interpretation (e.g., percentage-based)

#### Simple Fireball Variants
Added `spawn_simple_main_fireball` and `spawn_simple_secondary_fireball`:
- Original implementation without UV zoom or S-curve alpha
- Useful for other explosion effects that don't need UE5-accurate behavior

#### Dust Emitter Rework (Critical Fix)
Fixed dust emitter to match UE5 behavior:
- **Texture**: Replaced placeholder with correct UE5 T_Dirt_1.PNG (1024×1024, 4×1 flipbook)
- **Animation**: Changed from sequential frame animation to **random fixed frame** at spawn
  - UE5 uses `SubUV Animation Mode = Random` (NewEnumerator2)
  - Each particle picks ONE random frame (0-3) that stays fixed for lifetime
  - Gives visual variety without animation overhead
- **Size**: 1× UE5 spec (3-5m base), grows with (3×, 2×) scale curve
- **Spawn**: Exact origin (no offset), 2-3 particles
- **Alpha**: S-curve fade from 3.0→0 using smoothstep (brightness multiplier)
- **Color**: UE5 dark brown (0.147, 0.114, 0.070)
- **Velocity**: 35° narrow cone, 5-10 m/s upward
- **Lifetime**: 0.1-0.5s (very short, per UE5 spec)

#### Smoke Emitter Improvements
Fixed smoke emitter to match T3D-verified UE5 behavior:
- **XY Velocity**: Increased spread from ±8m to ±12m (1.5× UE5 spec) for better coverage relative to explosion size
- **Alpha Curve**: Changed from piecewise linear to **smoothstep bell curve** (0→0.5→0)
  - Uses cubic S-curve: `x * x * (3.0 - 2.0 * x)`
  - Ease-in to peak at t=0.5, ease-out from peak
  - Holds opacity longer near edges vs simple parabola
- **Color Curve**: Updated to T3D-verified values
  - Start: Dark brown RGB(0.147, 0.117, 0.089) = #261D17
  - End: Tan RGB(0.328, 0.235, 0.156) = #543C28
- **Lifetime**: Reduced from 1-4s to 0.8-2.5s for faster fade-out

## Next Steps

1. ~~Add dirt001 velocity-aligned debris emitter~~ ✅ Done
2. ~~Implement UE5-accurate color/alpha curves for dirt~~ ✅ Done
3. ~~Implement UE5-accurate dust/wisp emitters with brightness multiplier~~ ✅ Done
4. ~~Implement spark emitter with UE5-accurate HDR colors and flickering~~ ✅ Done
5. ~~Add sound effects integration~~ ✅ Done
6. Add collision detection for sparks (bounce once, then die)
7. Improve F8 Impact Flash ground alignment
