# Devlog 012: War FX Explosion Implementation Summary

**Date**: 2025-11-28
**Branch**: `feat/warfx-explosion`
**Status**: Working baseline implementation

## Overview

This devlog summarizes the implementation of Unity War FX explosion effects in Bevy 0.15. The system recreates the "Explosion" emitter from the War FX asset pack, which produces both flame bursts AND smoke puffs from a single particle system.

For detailed debugging history, see `devlogs/wfx_explosion/001-007*.md`.

---

## What Was Implemented

### 1. Additive Glow Effect (`wfx_additive.wgsl`)

A center glow that creates bright, soft-edged halos using additive blending.

**Key technique**: Luminance-based alpha with smoothstep for soft edges
```wgsl
let luminance = dot(tex.rgb, vec3<f32>(0.299, 0.587, 0.114));
let soft_alpha = smoothstep(0.0, 0.1, luminance);
let final_color = vec4<f32>(tex.rgb * tint.rgb * 4.0, soft_alpha * tint.a);
```

**Limitation discovered**: Bevy's depth prepass is NOT available for transparent/additive materials (`AlphaMode::Add`). Unity's depth-based soft particles are impossible in Bevy 0.15. Worked around with texture-based soft edges instead.

### 2. Smoke Scroll Shader (`wfx_smoke_scroll.wgsl`)

The main explosion shader that creates both flame and smoke from billboard quads.

**Unity's approach** (from `WFX_S Smoke Scroll.shader`):
```hlsl
float mask = tex2D(_MainTex, i.uv).a * i.color.a;
tex = lerp(fixed4(0.5,0.5,0.5,0.5), tex, mask);
// Blend DstColor SrcAlpha
```

The `lerp(0.5, color, mask)` is critical for multiply blend:
- mask=0 (edges): output 0.5 (neutral, no scene change)
- mask=1 (center): output tinted color
- RGB > 0.5 brightens scene, RGB < 0.5 darkens

---

## Critical Technical Discoveries

### 1. Bevy vs Unity Blend Equation Mismatch

**Bevy's `AlphaMode::Multiply`**:
```
final = dst * src.rgb + dst * (1 - src.a)
      = dst * (src.rgb + 1 - src.a)
```

**Unity's `Blend DstColor SrcAlpha`**:
```
final = dst * src.rgb + dst * src.a
      = dst * (src.rgb + src.a)
```

The difference: `(1 - src.a)` vs `src.a`. This broke Unity's `lerp(0.5, color, mask)` formula in Bevy.

**Solution**: Override blend state via `Material::specialize()`:
```rust
fn specialize(...) -> Result<(), SpecializedMeshPipelineError> {
    if let Some(ref mut fragment) = descriptor.fragment {
        for target in fragment.targets.iter_mut().flatten() {
            target.blend = Some(BlendState {
                color: BlendComponent {
                    src_factor: BlendFactor::Dst,
                    dst_factor: BlendFactor::SrcAlpha,  // Match Unity
                    operation: BlendOperation::Add,
                },
                alpha: BlendComponent::OVER,
            });
        }
    }
    Ok(())
}
```

### 2. Multiply Blend Math for Brightening/Darkening

With blend equation `dst * (rgb + alpha)`:
- **Brightening**: `rgb + alpha > 1.0`
- **Darkening**: `rgb + alpha < 1.0`
- **Neutral**: `rgb + alpha = 1.0`

Orange `(1.0, 0.522, 0.2)` only brightens red channel! Green and blue get darkened.

**Solution**: Boost center colors toward warm white `(1.0, 0.95, 0.85)` for fusing effect:
```wgsl
let center_boost = tex_alpha * particle_alpha;
let bright_flame = mix(darkened_flame, vec3<f32>(1.0, 0.95, 0.85), center_boost * 0.7);
```

### 3. Unity's Color System

Unity creates both flames AND smoke from ONE emitter via random start colors:

**Start Color Gradient** (particles randomly sample at spawn):
| Position | RGB | Visual |
|----------|-----|--------|
| 9% | (1.0, 0.522, 0.2) | Bright orange (flame) |
| 42% | (0.663, 0.235, 0.184) | Dark brown (smoke) |

**Color Over Lifetime** (multiplied over particle life):
| Time | Multiplier |
|------|------------|
| 0% | 1.0 |
| 20% | 0.694 |
| 41% | 0.404 (darkest) |
| 100% | 0.596 |

---

## File Structure

### Shaders
- `assets/shaders/wfx_additive.wgsl` - Additive glow with soft edges
- `assets/shaders/wfx_smoke_scroll.wgsl` - Main smoke/flame shader

### Rust Modules
- `src/wfx_materials.rs` - Material definitions with custom blend states
  - `SmokeScrollMaterial` - Multiply blend with Unity-matched equation
  - `AdditiveMaterial` - Additive blend for glow
- `src/wfx_spawn.rs` - Billboard spawning with Unity-matched parameters
  - 38 particles in 4 bursts (15+10+8+5)
  - Lifetime 3-4 seconds
  - Size 4-6 units

### Resources
- `resources/wfx_explosivesmoke_big/EXPLOSION_EMITTER_DETAILS.md` - Unity prefab analysis
- `devlogs/wfx_explosion/` - Detailed debugging history (7 devlogs)

---

## Current Shader State

```wgsl
// wfx_smoke_scroll.wgsl - key parts

// Sample texture alpha (high at center, low at edges)
let tex_alpha = textureSample(smoke_texture, smoke_sampler, uv).a;
let mask = tex_alpha * particle_alpha;

// Unity colors
let base_flame_color = vec3<f32>(1.0, 0.522, 0.2) * tint_color;
let base_smoke_color = vec3<f32>(0.663, 0.235, 0.184);

// Color Over Lifetime curve (piecewise linear)
let lifetime_t = 1.0 - particle_alpha;
let lifetime_mult = select(...);  // 1.0 → 0.694 → 0.404 → 0.596

// Boost center toward white for fusing
let center_boost = tex_alpha * particle_alpha;
let bright_flame = mix(darkened_flame, vec3<f32>(1.0, 0.95, 0.85), center_boost * 0.7);

// Spatial blend: center=flame, edges=smoke
let spatial_blend = tex_alpha * (0.5 + 0.5 * particle_alpha);
let pixel_color = mix(darkened_smoke, bright_flame, spatial_blend);

// Unity's lerp formula
let final_rgb = mix(vec3<f32>(0.5), pixel_color, mask);
let final_alpha = mix(0.5, target_alpha, mask);
```

---

## What Works

- Bright fusing cores when particles overlap
- Orange/red flame color (not pure white)
- Dark smoke at edges and toward end of lifetime
- Smooth particle edges (no rectangular borders)
- Unity-matched particle counts and timing

## Not Yet Implemented

- UV scrolling animation (texture morphing)
- Random start color sampling (currently all particles same color)
- Sound effects integration
- Integration with game combat system

---

## Key Commits

- `9df2f09` - Custom blend state via `Material::specialize()`
- `c453599` - Spatial color blend (center=bright, edges=smoke)
- `3b729f6` - Unity-accurate colors and fusing cores

## Testing

Press `2` key to spawn explosion at center position (0, 10, 0).

---

## Lessons Learned

1. **Don't assume engine parity** - Unity features (depth-based soft particles) may be architecturally impossible in Bevy
2. **Read engine source code** - Found Bevy's actual blend equation in `bevy_pbr::render::mesh.rs`
3. **Understand the math** - Multiply blend `dst * (rgb + alpha)` requires specific RGB+alpha values for intended effects
4. **`Material::specialize()` is powerful** - Can override blend state, depth settings, etc. at pipeline level

---

## Retrospective: The Debugging Journey

### What Made This Hard

This implementation spanned ~5 Claude Code threads over multiple sessions. The difficulty came from:

1. **Invisible intermediate states** - Shader bugs often resulted in completely black or invisible output, making it hard to know what was wrong
2. **Multiple interacting systems** - Blend mode, texture alpha, color values, and lifetime curves all interact; changing one breaks others
3. **Engine abstraction differences** - Unity's `Blend DstColor SrcAlpha` has no direct Bevy equivalent
4. **Undocumented behavior** - Had to read Bevy source code to find actual blend equations

### The Major Struggles

#### 1. Rectangular Border Problem (2+ sessions)

**Symptom**: Particle quads showed visible rectangular edges instead of soft smoke shapes.

**Wrong assumptions tried**:
- Thought texture alpha was wrong → wrote Python script to analyze TGA file
- Thought UV mapping was broken → created custom quad mesh with explicit UVs
- Thought alpha inversion was needed → inverted texture alpha in shader
- Tried radial falloff overlay → created ugly artifacts

**Actual cause**: Bevy's `AlphaMode::Multiply` uses `dst * (src.rgb + 1 - src.a)`, not Unity's `dst * (src.rgb + src.a)`. The `lerp(0.5, color, mask)` formula only works with Unity's blend equation.

**Solution discovery**: Had to dig into `~/.cargo/registry/src/.../bevy_pbr-0.15.3/src/render/mesh.rs` lines 1791-1800 to find the actual blend state, then override it via `Material::specialize()`.

#### 2. Dark Core / Bright Rim Inversion (1+ sessions)

**Symptom**: Particles had dark centers and bright edges - exact opposite of desired effect.

**Debugging steps**:
- Added debug shader to visualize raw texture alpha as grayscale
- Discovered Bevy reads TGA alpha inverted from what Python/PIL shows
- Tried inverting alpha in shader → partially fixed but created new issues

**Root cause**: Combination of alpha interpretation AND blend equation issues. The "fix" of inverting alpha was actually a workaround that masked the real blend equation problem.

#### 3. Missing Flame Effect - Only Smoke Visible (1 session)

**Symptom**: Only dark smoke appeared, no bright orange flame burst.

**Wrong assumption**: Thought the shader color logic was wrong.

**Actual cause**: In `animate_explosion_billboards()`, we were premultiplying alpha into the color:
```rust
// BUG: Darkens color prematurely
material.tint_color_and_speed = Vec4::new(
    current_color.x * current_alpha,  // ← Wrong!
    ...
);
```

**Fix**: Pack alpha separately into the w component alongside scroll_speed.

#### 4. White Cores Instead of Orange (1 session)

**Symptom**: Flame cores were pure white instead of orange/red.

**Cause**: Shader was pushing flame color 80% toward white:
```wgsl
var flame_color = mix(tint_color, vec3<f32>(1.0), 0.8);  // Too white!
```

**Additional discovery**: Even with correct orange `(1.0, 0.522, 0.2)`, the multiply blend math means orange only brightens the red channel - green and blue get darkened! Had to boost center toward warm white `(1.0, 0.95, 0.85)` to get proper fusing effect.

### Mistakes Made

1. **Assumed Bevy's AlphaMode::Multiply matched Unity** - Wasted time trying workarounds before checking source
2. **Over-complicated early** - Added radial falloffs, power curves, multiple masks before understanding the core issue
3. **Didn't visualize intermediate values sooner** - Debug shaders showing raw texture/alpha would have saved hours
4. **Premultiplied alpha in CPU code** - Classic VFX mistake; alpha should be handled in shader

### What Finally Worked

The breakthrough came from:
1. **Reading Bevy's source code** to find the actual blend equation
2. **Using `Material::specialize()`** to override blend state at pipeline level
3. **Extracting exact values from Unity** - parsed prefab YAML for precise color gradients
4. **Understanding multiply blend math** - `rgb + alpha > 1.0` brightens, `< 1.0` darkens

### Time Investment

Rough estimate across all sessions:
- Additive glow shader: ~2-3 hours (relatively straightforward)
- Smoke scroll shader debugging: ~8-10 hours (most of the struggle)
- Color tuning and fusing: ~2 hours (after blend was fixed)
- Documentation and cleanup: ~1-2 hours

Total: ~15+ hours for what is essentially one particle effect with two shaders.

### Was It Worth It?

Yes, because:
1. **Learned Bevy's rendering internals** - `Material::specialize()`, blend states, pipeline descriptors
2. **Documented the solutions** - Future Unity→Bevy VFX ports can reference this
3. **Quality baseline established** - The effect actually looks close to the Unity original
4. **Reusable patterns** - The custom blend state approach applies to other effects

---

## Summary for Future Reference

If you're porting Unity VFX to Bevy:

1. **Check blend equations first** - Unity's blend modes don't map 1:1 to Bevy's AlphaMode
2. **Use `Material::specialize()`** - You can override any pipeline state including blend
3. **Extract exact values from Unity** - Parse prefab/material files for precise numbers
4. **Debug with visualization shaders** - Output intermediate values as colors to see what's happening
5. **Understand the math** - VFX is all about blend equations, color spaces, and interpolation curves
