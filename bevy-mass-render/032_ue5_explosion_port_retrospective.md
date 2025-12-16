# Devlog 032: UE5 Explosion Port Retrospective

## Overview

This devlog captures the lessons learned from porting UE5's `NS_Explosion_Sand_5` Niagara particle system to Bevy. The process revealed several non-obvious challenges that would apply to any UE5 → Bevy VFX port.

## The Hardest Problems

### 1. SubUV Animation Mode: Random vs Sequential

**The Problem**: The dust emitter looked completely wrong - particles were "flashing" and the effect was sparse and jittery.

**Root Cause**: UE5's Niagara has a `SubUV Animation Mode` parameter with multiple options:
- `Linear` (default): Sequential frame animation 0→1→2→3...
- `Random` (NewEnumerator2): Each particle picks ONE random frame at spawn, stays fixed

The dust emitter uses **Random** mode - each particle shows a different static frame for visual variety, NOT an animation. The T3D export showed `SubUV Animation Mode = NewEnumerator2` which means Random.

**The Fix**: Changed from `FlipbookSprite { total_frames: 4, ... }` (animating) to `total_frames: 1` with a random `frame_col` at spawn.

**Lesson**: Always check `SubUV Animation Mode` in Niagara exports. "Random" is commonly used for debris/dust where variety matters more than animation.

### 2. Wrong Texture Files

**The Problem**: Multiple emitters looked wrong because placeholder textures didn't match UE5 originals.

**Examples**:
- Dirt: Was using T_Dirt_1.PNG instead of T_Dirt_3.PNG
- Dust: Was using a 256×256 placeholder instead of the correct 1024×1024 T_Dirt_1.PNG

**The Fix**: Directly copy textures from the UE5 project's Content folder. Don't assume naming conventions or create placeholders.

**Lesson**: Texture accuracy is critical. A 4×1 flipbook with wrong sprites will never look right regardless of code quality.

### 3. Alpha Curves: Simple vs Smoothstep

**The Problem**: Particles faded in/out too abruptly or held opacity too long.

**Root Cause**: UE5's Niagara curves use cubic interpolation (RCIM_Cubic) by default. The exported LUT values don't map to simple linear interpolation.

**Example - Smoke Alpha**:
```
Documented (WRONG - simple parabola):
t=0.1: 0.18    t=0.25: 0.38

T3D Actual (smoothstep bell):
t=0.1: 0.05    t=0.25: 0.24
```

**The Fix**: Use smoothstep `x * x * (3.0 - 2.0 * x)` for fade curves instead of linear interpolation.

**Lesson**: When UE5 shows `InterpMode=RCIM_Cubic`, the curve is NOT linear. Verify against LUT/ShaderLUT values if available.

### 4. Alpha > 1.0 as Brightness Multiplier

**The Problem**: Dust and wisp emitters looked too dim compared to UE5.

**Root Cause**: UE5's Niagara uses alpha values > 1.0 as brightness multipliers. An alpha of 3.0 means "3× brighter RGB, but still fully opaque."

**The Fix**: Modified the shader to handle this:
```wgsl
let alpha_multiplier = color_data.a;
let tinted_color = sprite_sample.rgb * color_data.rgb * alpha_multiplier;
let final_alpha = sprite_sample.a * min(alpha_multiplier, 1.0);
```

**Lesson**: UE5 particle alpha is not just opacity - values > 1.0 boost RGB intensity. This is how initial "bright flash" effects work.

### 5. Local Space Velocity

**The Problem**: Smoke spread looked wrong, moving in world space instead of relative to the camera.

**Root Cause**: UE5's `bLocalSpace = True` means velocity is relative to the emitter's orientation. For camera-facing particles, "local X/Y" means screen-space spread.

**The Fix**: Transform velocity using camera orientation:
```rust
let velocity = cam_right * local_x + cam_up * local_y + cam_forward * local_z;
```

**Lesson**: Always check `bLocalSpace` in Niagara exports. If true, velocity axes are emitter-relative, not world-relative.

### 6. UV Zoom Effect

**The Problem**: UE5 fireball has a "zoom out" effect where UV scale goes from 500→1 over lifetime. Direct implementation made the effect invisible.

**Root Cause**: UE5's UV scale interpretation may be percentage-based or work differently than simple UV division. A 500× zoom would show a nearly invisible center pixel.

**Current Status**: Disabled. Needs investigation of UE5's material shader to understand the actual behavior.

**Lesson**: Some UE5 features don't translate directly. When documentation says "500→1 UV scale", verify what that actually means in the material shader.

### 7. Scale Curves: Growth vs Shrink

**The Problem**: Dirt001 particles were shrinking when they should grow.

**Root Cause**: Different emitters have opposite scale behaviors:
- Dirt (billboard): Shrinks 100→0 over lifetime
- Dirt001 (velocity-aligned): Grows 1→2× over lifetime

Easy to confuse when copying code between similar emitters.

**Lesson**: Each emitter has unique curves. Don't assume similar emitters have similar behaviors.

### 8. System Conflicts (Alpha Override)

**The Problem**: Dust particles were "flashing" despite correct alpha curve code.

**Root Cause**: Two systems were fighting:
- `animate_flipbook_sprites`: Updated alpha based on flipbook timing
- `update_dust_alpha`: Applied the correct dust alpha curve

The flipbook system was overwriting dust alpha every frame.

**The Fix**: Added `is_dust` check to skip alpha updates in `animate_flipbook_sprites` for particles with `DustScaleOverLife`.

**Lesson**: When multiple systems modify the same data (alpha, color, scale), ensure they don't conflict. Use component markers to route updates correctly.

## What Worked Well

### T3D Export + Documentation

Exporting UE5's Niagara system to T3D and creating detailed markdown documentation for each emitter was invaluable. The docs captured:
- Exact parameter values
- Curve keyframes with interpolation modes
- Material node graphs
- LUT samples for verification

### Iterative Visual Comparison

Running both UE5 and Bevy side-by-side made discrepancies obvious. Each "that looks wrong" moment led to discovering a new gotcha.

### Agent-Assisted Verification

Using AI agents to cross-check documentation against T3D source data caught several curve formula errors that manual inspection missed.

## Key Takeaways for Future Ports

1. **Export to T3D**: Machine-readable format captures everything
2. **Check SubUV Animation Mode**: Random vs Sequential is a common gotcha
3. **Use exact textures**: Copy from Content folder, don't substitute
4. **Verify curve interpolation**: Cubic ≠ Linear
5. **Alpha > 1.0 = brightness boost**: Common UE5 pattern
6. **Check bLocalSpace**: Velocity reference frame matters
7. **Test each emitter solo**: Find problems before combining
8. **Watch for system conflicts**: Multiple systems modifying same data

## Final Result

The port successfully recreates the UE5 ground explosion effect with all major emitters:
- Main/Secondary fireballs with S-curve alpha fade
- Dirt/Dirt001 debris with physics and scale curves
- Dust ring with random fixed frames and brightness multiplier
- Wisps with gravity arc motion
- Smoke cloud with smoothstep bell alpha curve
- Sparks with HDR cooling ember effect
- Flash sparks with shooting star XY scale
- 3D mesh parts with bounce physics

The effect is visually close to UE5 while being tuned for the game's specific needs (shorter smoke lifetime, adjusted spark heights, etc.).
