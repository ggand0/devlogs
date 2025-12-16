# Devlog 001: WFX Smoke Scroll Shader - Debugging Brightness and Edge Issues

**Date**: 2025-11-27
**Branch**: `feat/warfx-explosion`
**Latest WIP Commit**: `c244c77`

## Goal

Reproduce Unity's "Explosion" emitter from the War FX asset using Bevy's custom materials and WGSL shaders. The key visual requirements:

1. **Bright fusing cores** - overlapping particles should blend together creating bright white centers
2. **Soft transparent edges** - particle quads should not show rectangular borders
3. **Flame→smoke transition** - bright at start, dark smoke at end of lifetime

## Key Discovery: Texture Alpha is Inverted in Bevy

### Python Analysis vs Runtime Behavior

**Python analysis of `WFX_T_SmokeLoopAlpha.tga`:**
```
Center (128, 128): alpha = 239 (HIGH)
Corner (5, 5): alpha = 0 (LOW)
Edge (128, 5): alpha = 0 (LOW)
```

**Debug shader output in Bevy** (rendering `tex_alpha` as grayscale):
- Center: transparent (LOW alpha displayed)
- Edges/Corners: solid black (HIGH alpha displayed)

**Conclusion**: Bevy reads the TGA texture alpha **inverted** from what Python/PIL shows. This is likely a format interpretation issue (premultiplied alpha, gamma, or TGA format flags).

### Solution: Invert Alpha in Shader

```wgsl
let raw_alpha = textureSample(smoke_texture, smoke_sampler, uv).a;
let tex_alpha = 1.0 - raw_alpha;  // Invert to match Unity's expected behavior
```

Or equivalently, use `spatial_mask = 1.0 - tex_alpha` where `tex_alpha` is the raw sample.

## Unity Shader Logic (Reference)

From `WFX_S Smoke Scroll.shader`:

```hlsl
float mask = tex2D(_MainTex, i.uv).a * i.color.a;
i.uv.y -= fmod(_Time*_ScrollSpeed,1);  // UV scrolling
fixed4 tex = tex2D(_MainTex, i.uv);
tex.rgb *= i.color.rgb * _TintColor.rgb;
tex.a = mask;
tex = lerp(fixed4(0.5,0.5,0.5,0.5), tex, mask);  // KEY LINE
```

**Blend mode**: `Blend DstColor SrcAlpha` (multiply blend)

**How it works**:
- `lerp(0.5, color, mask)` blends toward gray at edges
- For multiply blend: RGB > 0.5 brightens scene, RGB < 0.5 darkens
- Where mask=0 (edges): output 0.5 (neutral, no change)
- Where mask=1 (center): output tinted color

## Current Shader State

**File**: `assets/shaders/wfx_smoke_scroll.wgsl`

```wgsl
// Invert texture alpha (Bevy reads it inverted from file)
let spatial_mask = 1.0 - tex_alpha;

// Flame→smoke color transition based on particle lifetime
let bright_color = mix(tint_color, vec3(1.0), 0.8);  // 80% toward white
let smoke_color = vec3(0.3);  // Dark smoke
let lifetime_color = mix(smoke_color, bright_color, particle_alpha);

// Apply spatial mask for brightness distribution
let final_rgb = mix(vec3(0.5), lifetime_color, spatial_mask);

// Alpha for visibility (with slower fade)
let fade_alpha = pow(particle_alpha, 0.3);
let final_alpha = spatial_mask * fade_alpha;
```

## Issues Fixed

### 1. Dark Core / Bright Rim (FIXED)
**Cause**: Texture alpha inverted in Bevy
**Fix**: Use `spatial_mask = 1.0 - tex_alpha`

### 2. Entire Quad Gets Bright at End of Lifetime (FIXED)
**Cause**: As `particle_alpha → 0`, the mask decreased everywhere, making edges approach 0.5 (bright for multiply blend)
**Fix**: Separate spatial distribution from lifetime fade. Use `spatial_mask` for brightness pattern, `particle_alpha` only for color transition (flame→smoke).

### 3. Particles Get Bright Instead of Dark Smoke at End (FIXED)
**Cause**: Low alpha made particles invisible, showing background
**Fix**: Use `pow(particle_alpha, 0.3)` for slower alpha fade, and transition RGB to dark smoke color (0.3) which darkens the scene.

## Outstanding Issue

### Rectangular Quad Borders Visible

The quad corners/edges are still somewhat visible. Analysis:

**Texture alpha pattern** (from Python):
```
y=  0%:   0   0   0   0   0
y= 25%:   0  24 142  49   0
y= 50%:   0 125 239 127   0
y= 75%:   0  12 119   7   0
y=100%:   0   0   0   0   0
```

The texture is a **center blob** with alpha falloff, not a full-quad coverage. Corners have alpha=0.

With Bevy's inversion:
- Corner tex_alpha ≈ 1.0 (inverted from 0)
- `spatial_mask = 1.0 - 1.0 = 0` at corners

This should make corners transparent, but there may be edge cases or the falloff isn't smooth enough.

**Potential fixes to try**:
1. Pre-process texture to ensure proper alpha channel
2. Add artificial radial falloff in shader
3. Check if texture wrap mode affects edge sampling

## Files Modified

- `assets/shaders/wfx_smoke_scroll.wgsl` - Main shader with all fixes
- `src/wfx_materials.rs` - `AlphaMode::Multiply` for blend mode
- `src/wfx_spawn.rs` - Custom quad mesh with explicit UVs

## Commits

- `c244c77` - WIP: Smoke scroll shader with bright core, dark rim, flame→smoke transition

## Next Steps

1. Investigate why rectangular borders are still visible
2. Re-enable UV scrolling once brightness issues are fully resolved
3. Add center glow effect (separate additive emitter)
4. Tune color curves to match Unity's exact gradients
