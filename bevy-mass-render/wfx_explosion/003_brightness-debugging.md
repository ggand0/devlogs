# Devlog 014: WFX Explosion Brightness Debugging

**Date**: 2025-11-27
**Issue**: Dark core / bright rim - inverted from expected behavior

## Problem Summary

The explosion particles show inverted brightness:
- **Expected**: Bright center that fuses when particles overlap, soft transparent edges
- **Actual**: Dark core, bright rim around edges

## Background: Unity Shader Logic

The original Unity shader (`WFX_S Smoke Scroll.shader`) uses:
```hlsl
float mask = tex2D(_MainTex, i.uv).a * i.color.a;
i.uv.y -= fmod(_Time*_ScrollSpeed,1);
fixed4 tex = tex2D(_MainTex, i.uv);
tex.rgb *= i.color.rgb * _TintColor.rgb;
tex.a = mask;
tex = lerp(fixed4(0.5,0.5,0.5,0.5), tex, mask);  // KEY LINE
```

The `lerp(0.5, tex, mask)` is critical for multiply blend (`Blend DstColor SrcAlpha`):
- Where mask=0 (edges): output 0.5 (neutral, no change to scene)
- Where mask=1 (center): output tinted texture color
- Values > 0.5 brighten the scene, values < 0.5 darken it
- Overlapping particles with values > 0.5 "fuse" to create bright white cores

## What We've Tried

### 1. Texture Analysis (Python)
Sampled WFX_T_SmokeLoopAlpha.tga:
- Center pixel: RGB=133 (0.52), Alpha=239 (0.94)
- Corner pixel: RGB=150 (0.59), Alpha=0
- Edge pixel: RGB=176 (0.69), Alpha=0

**Conclusion**: Texture alpha IS high at center, low at edges. But RGB is DARKER at center than edges.

### 2. Shader Approaches Tried

#### A. Original Unity formula with lerp
```wgsl
let mask = tex_alpha * particle_alpha;
let tinted = tex.rgb * tint_color;
let final_rgb = mix(vec3(0.5), tinted, mask);  // Unity's lerp formula
return vec4(final_rgb, mask);
```
**Result**: Dark core because tex.rgb is dark at center (0.52), and after tinting with orange (1.0, 0.52, 0.2), the G and B channels are < 0.5, causing darkening.

#### B. Boost texture brightness before lerp
```wgsl
let boosted_tex = tex.rgb * 1.6;  // 0.52 * 1.6 = 0.83
var tinted_rgb = boosted_tex * tint_color;
let final_rgb = mix(vec3(0.5), tinted_rgb, mask);
```
**Result**: Still dark core - after orange tint, G=0.43, B=0.17 which are < 0.5

#### C. Ignore texture RGB, use tint directly with lerp
```wgsl
var bright_tint = tint_color * 1.4;
let final_rgb = mix(vec3(0.5), bright_tint, mask);
```
**Result**: Still dark core - suggests mask itself is inverted from what we expect

#### D. Blend toward white based on mask (enhanced lerp)
```wgsl
let white_amount = mask * 0.6;
var bright_color = mix(tint_color, vec3(1.0), white_amount);
bright_color = max(bright_color, vec3(0.55));  // Ensure > 0.5
let final_rgb = mix(vec3(0.5), bright_color, mask);
```
**Result**: Still dark core, bright rim - the mask must be inverted

#### E. Invert mask for brightness
```wgsl
let mask = (1.0 - tex_alpha) * particle_alpha;
```
**Result**: Bright core initially, but harsh edges (quad borders visible)

#### F. Separate mask for brightness vs alpha for edges
```wgsl
let mask = (1.0 - tex_alpha) * particle_alpha;  // For brightness calculation
let soft_alpha = tex_alpha * particle_alpha;     // For edge softness
let final_rgb = mix(vec3(0.5), bright_color, mask);
return vec4(final_rgb, soft_alpha);
```
**Result**: Even darker core - the separation doesn't work as expected

## Key Observations

1. When we inverted the mask, we briefly saw bright core → dark (phase 1) then bright everywhere (phase 2)
2. The texture alpha sampling might not be working correctly with our custom mesh
3. Debug attempts to visualize UVs and texture showed all black, suggesting texture/UV issues

## Current State

- Using `AlphaMode::Multiply` (correct for Unity's `Blend DstColor SrcAlpha`)
- Custom quad mesh with explicit UVs created but may not be working
- Shader has inverted mask for brightness + original alpha for edge softness
- Still getting dark core, bright rim

## Next Steps to Try

1. **Verify texture is actually loading** - try with StandardMaterial first
2. **Check if UV coordinates work** - the debug output showed black, suggesting UVs might be 0
3. **Try different blend mode** - maybe Bevy's Multiply doesn't work like Unity's
4. **Simplify** - just output solid color based on tint_color to verify shader runs

## Files Modified

- `assets/shaders/wfx_smoke_scroll.wgsl` - Multiple iterations
- `src/wfx_materials.rs` - AlphaMode changes
- `src/wfx_spawn.rs` - Added custom quad mesh with explicit UVs
- `src/objective.rs` - Disabled center glow for testing
