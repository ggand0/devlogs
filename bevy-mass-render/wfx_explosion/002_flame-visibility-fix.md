# Devlog 013: WFX Explosion Flame Visibility Fix

**Date**: 2025-11-27
**Issue**: Explosion emitter shows only dark smoke, no visible flame burst

## Problem Summary

When pressing `2` key to spawn the explosion effect, only dark smoke particles are visible with faint orange at the borders. The bright flame burst effect from the original Unity War FX asset is missing.

## Root Cause Analysis

### How Unity Creates Both Flames AND Smoke from One Emitter

The Unity "Explosion" emitter creates both visual elements from a **single particle system** through:

1. **Random Start Color from Gradient** - particles randomly sample colors at spawn:
   - Orange colors (gradient position ~9%) → appear as flames
   - Brown colors (gradient position ~42%) → appear as smoke

2. **Color Over Lifetime** - multiplies by grayscale to fade all particles to gray:
   - `final_color = StartColor × ColorOverLifetime`
   - Orange start → orange flame that fades to gray smoke
   - Brown start → smoke-colored throughout

3. **UV Scrolling + Multiply Blend** - creates volumetric, morphing appearance

### The Bug: Premultiplied Alpha

Found in Unity material `WFX_M_ExplosiveSmoke.mat`:
- `_TintColor = (1.0, 1.0, 1.0, 1.0)` - pure white, NOT the shader default (0.5, 0.5, 0.5)
- `_ScrollSpeed = 10.0`

Unity shader formula:
```hlsl
tex.rgb *= i.color.rgb * _TintColor.rgb;  // i.color = StartColor × ColorOverLifetime
tex = lerp(fixed4(0.5,0.5,0.5,0.5), tex, mask);
```

For orange particle at spawn with texture RGB ~0.64:
- `tinted = 0.64 * orange(1.0, 0.52, 0.2) * white(1.0, 1.0, 1.0)`
- `tinted = (0.64, 0.33, 0.13)`
- R channel = 0.64 > 0.5 → **brightens** red in scene (multiply blend)
- G,B < 0.5 → darkens green/blue → creates orange/red tint

**Our bug**: In `animate_explosion_billboards()`, we premultiply alpha into the color:
```rust
material.tint_color_and_speed = Vec4::new(
    current_color.x * current_alpha,  // ← Darkens prematurely!
    current_color.y * current_alpha,
    current_color.z * current_alpha,
    scroll_speed,
);
```

This makes particles dark from the start, before the multiply blend even happens.

## Solution

### Step 1: Remove alpha premultiplication

Pass color without premultiplying alpha. Pack alpha into the w component alongside scroll_speed:
```rust
// Pack: scroll_speed as integer + alpha as decimal
let packed = scroll_speed.floor() + current_alpha;
material.tint_color_and_speed = Vec4::new(
    current_color.x,
    current_color.y,
    current_color.z,
    packed,
);
```

### Step 2: Update shader to extract and apply alpha

In wfx_smoke_scroll.wgsl:
```wgsl
let scroll_speed = floor(material.tint_color_and_speed.a);
let particle_alpha = fract(material.tint_color_and_speed.a);

// Apply alpha to mask (like Unity: mask = tex.a * i.color.a)
let mask = textureSample(smoke_texture, smoke_sampler, uv).a * particle_alpha;
```

## Files Modified

1. `src/wfx_spawn.rs` - `animate_explosion_billboards()`
2. `assets/shaders/wfx_smoke_scroll.wgsl`

## Expected Result

- Orange particles at spawn have full brightness
- R channel > 0.5 creates visible orange/red tint via multiply blend
- Alpha fades particles over lifetime via mask modulation
- Color transitions to gray over lifetime for smoke appearance
