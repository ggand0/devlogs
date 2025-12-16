# Devlog 002: WFX Smoke Scroll Shader - Rectangular Border Issue

**Date**: 2025-11-27
**Branch**: `feat/warfx-explosion`
**Previous Devlog**: 001_smoke_scroll_shader_debugging.md

## Problem

Rectangular quad borders are clearly visible. The particles show their quad shape instead of blending smoothly into the scene.

## Debug Visualization Tests

### Test 1: Raw Texture Alpha as Grayscale
```wgsl
let tex = textureSample(smoke_texture, smoke_sampler, uv);
return vec4<f32>(tex.a, tex.a, tex.a, 1.0);
```
**Result**: "transparent pattern center and dark rim"
- Confirms Bevy reads alpha INVERTED (low at center, high at edges)
- This matches previous devlog findings

### Test 2: Raw Texture RGB
```wgsl
let tex = textureSample(smoke_texture, smoke_sampler, uv);
return vec4<f32>(tex.rgb, 1.0);
```
**Result**: "transparent / dark pattern"
- RGB channel shows grayscale smoke pattern
- Dark areas where smoke detail is

## Shader Approaches Tried (This Session)

### Approach A: Edge Softening with Power Curve
```wgsl
let spatial_mask = 1.0 - tex_alpha;
let edge_softness = pow(spatial_mask, 0.5);  // sqrt for softer edges
let final_alpha = edge_softness * fade_alpha;
```
**Result**: Did not fix rectangular borders

### Approach B: Radial Falloff Overlay
```wgsl
let centered_uv = uv - vec2<f32>(0.5);
let dist_from_center = length(centered_uv) * 2.0;  // 0 at center, ~1.41 at corners
let radial_falloff = 1.0 - smoothstep(0.8, 1.0, dist_from_center);
let combined_mask = spatial_mask * radial_falloff;
```
**Result**: "looks like shit - still rectangular shape, black circle line with some thickness, bright-ish core"
- The radial falloff created visible artifacts instead of hiding borders

### Approach C: Unity's Full vec4 Lerp
```wgsl
let corrected_alpha = 1.0 - tex.a;
let mask = corrected_alpha * particle_alpha;
let final_color = mix(vec3<f32>(0.5), base_color, mask);
let final_alpha = mix(0.5, 1.0, mask);  // Alpha also lerps to 0.5 at edges
```
**Result**: "rectangle quad where the core is transparent with some pattern and dark rim but overall more transparent than before"
- Using 0.5 alpha at edges doesn't hide borders - it just makes them semi-transparent gray

## Key Finding: Bevy's AlphaMode::Multiply Blend Equation

**Source**: `~/.cargo/registry/src/.../bevy_pbr-0.15.3/src/render/mesh.rs` lines 1791-1800

```rust
blend = Some(BlendState {
    color: BlendComponent {
        src_factor: BlendFactor::Dst,
        dst_factor: BlendFactor::OneMinusSrcAlpha,
        operation: BlendOperation::Add,
    },
    alpha: BlendComponent::OVER,
});
```

**Bevy's actual blend equation**:
```
final = dst * src.rgb + dst * (1 - src.a)
      = dst * (src.rgb + 1 - src.a)
```

**Unity's `Blend DstColor SrcAlpha`**:
```
final = dst * src.rgb * src.a + dst * (1 - src.a)  // (approximately)
```

### Critical Difference

For Bevy's multiply blend:
- **Neutral (no change)**: `src.rgb + 1 - src.a = 1.0` → requires `src.rgb = src.a`
- **Brightening**: `src.rgb + 1 - src.a > 1.0`
- **Darkening**: `src.rgb + 1 - src.a < 1.0`

This means:
| RGB | Alpha | Factor | Effect |
|-----|-------|--------|--------|
| 0.0 | 0.0 | 0 + 1 - 0 = 1.0 | **Neutral** (edges!) |
| 0.5 | 0.5 | 0.5 + 1 - 0.5 = 1.0 | Neutral |
| 0.5 | 1.0 | 0.5 + 1 - 1 = 0.5 | Darkens |
| 1.0 | 1.0 | 1.0 + 1 - 1 = 1.0 | Neutral |
| 1.5 | 1.0 | 1.5 + 1 - 1 = 1.5 | Brightens |

### Solution for Soft Edges

For truly invisible edges in Bevy's multiply blend:
```wgsl
// At edges (mask=0): rgb=0, alpha=0 → factor = 1 (neutral!)
// At center (mask=1): rgb=color, alpha=1 → factor = color
let final_rgb = base_color * mask;
let final_alpha = mask;
```

This is different from Unity's `lerp(0.5, color, mask)` approach!

## Questions Answered

1. **What is Bevy's exact blend equation?** → `dst * (src.rgb + 1 - src.a)`
2. **Why doesn't 0.5 work at edges?** → Because `0.5 + 1 - 0.5 = 1.0` only works if rgb=alpha. With `lerp(0.5, ..., 0)` we get rgb=0.5, alpha=0.5 which IS neutral, but only accidentally.
3. **Why did edges still show?** → The mask wasn't reaching exactly 0 at edges due to texture sampling

## Current Shader State

```wgsl
// Sample texture - Bevy reads alpha INVERTED (low at center, high at edges)
let tex = textureSample(smoke_texture, smoke_sampler, uv);

// Correct the inversion: mask should be HIGH at center, LOW at edges
let corrected_alpha = 1.0 - tex.a;

// Unity formula: mask = tex_alpha * vertex_alpha
let mask = corrected_alpha * particle_alpha;

// Bright flame→dark smoke transition over lifetime
var flame_color = mix(tint_color, vec3<f32>(1.0), 0.7);
flame_color = max(flame_color, vec3<f32>(0.8));
let smoke_color = vec3<f32>(0.35);
let base_color = mix(smoke_color, flame_color, particle_alpha);

// Unity's key formula: lerp(0.5, color, mask)
let final_color = mix(vec3<f32>(0.5), base_color, mask);
let final_alpha = mix(0.5, 1.0, mask);

return vec4<f32>(final_color, final_alpha);
```

## Additional Failed Approaches

### Approach D: Bevy-Correct Neutral Edges (rgb=0, alpha=0)
```wgsl
let final_rgb = base_color * mask;
let final_alpha = mask;
```
**Result**: "transparent core, no bright explosion flame effect, overall just faded/dull garbage"
- Mathematically correct for Bevy's blend equation but loses the flame effect entirely

### Approach E: Switch to AlphaMode::Add
```wgsl
// Additive blend for bright fusing cores
var flame_color = mix(tint_color, vec3<f32>(1.0, 0.9, 0.7), 0.6);
let intensity = particle_alpha * particle_alpha;
let final_color = flame_color * intensity;
let final_alpha = tex_mask * particle_alpha;
```
**Result**: "pretty shitty in another way" - different visual style, not the desired effect

## Conclusion

**SOLVED in [003_custom_blend_state_solution.md](003_custom_blend_state_solution.md)**

The fix was option #3: Custom blend state via `Material::specialize()` to match Unity's exact `Blend DstColor SrcAlpha`. Also required removing the alpha inversion in the shader.
