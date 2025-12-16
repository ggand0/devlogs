# Devlog 013: War FX Combined Explosion - Final Tuning & Integration

**Date**: 2025-11-28
**Branch**: `feat/warfx-explosion` (merged to main)
**PR**: Add War FX explosion effect system

## Overview

This session completed the War FX explosion implementation by adding the final two emitters (dot sparkles), fixing visual issues, and integrating the combined effect into the tower destruction system.

---

## What Was Implemented

### Complete 6-Emitter Explosion System

The combined explosion (`spawn_combined_explosion`) now includes all emitters from Unity's WFX_ExplosiveSmoke_Big prefab:

| Emitter | Particles | Key Behavior |
|---------|-----------|--------------|
| Embedded glow | 2 | Central additive flash, 0.7s lifetime |
| Explosion flames | 57 | UV-scrolling smoke/fire billboards, spherical spawn |
| Smoke emitter | 30 | Delayed 0.5s, gray smoke particles |
| Glow sparkles | 35 | Fast embers with gravity (4 units/sec²) |
| **Dot sparkles** | 75 | Dense sparks, lighter gravity (2 units/sec²) |
| **Dot sparkles vertical** | 15 | Upward-floating, zero gravity |

**Total**: ~214 particles per explosion

### Dot Sparkles Implementation

Two new emitter functions added:

```rust
// Dense shower of bright sparks - spray outward with gravity
spawn_dot_sparkles(commands, meshes, materials, asset_server, position, scale);

// Rising sparks that float upward (no gravity)
spawn_dot_sparkles_vertical(commands, meshes, materials, asset_server, position, scale);
```

Both use the GlowCircle texture instead of SmallDots atlas (the atlas texture was nearly invisible at small particle scales).

---

## Key Fixes This Session

### 1. Glow Hard-Cut Fix (Root Cause)

**Problem**: Glow effect had abrupt cutoff at end of lifetime instead of smooth fade.

**Failed attempts**:
- More keyframes in alpha curve
- Extended lifetime
- Despawn buffer hack

**Root cause**: In `wfx_additive.wgsl`, RGB output wasn't multiplied by `tint_color.a`. With additive blending, alpha=0 but RGB>0 still produces visible output.

**Fix**:
```wgsl
let final_color = vec4<f32>(
    tex.rgb * tint_color.rgb * brightness * soft_alpha * tint_color.a,  // Added * tint_color.a
    final_alpha
);
```

### 2. Sparkle Size Not Scaling

**Problem**: Glow sparkles weren't respecting the global scale parameter.

**Fix**: Added `* scale` to size curve keyframes:
```rust
let scale_curve = AnimationCurve {
    keyframes: vec![
        (0.0, 0.05 * size_mult * scale),   // Was missing * scale
        (0.347, 0.095 * size_mult * scale),
        (1.0, 0.123 * size_mult * scale),
    ],
};
```

### 3. Flame Distribution

**Problem**: Flames clustered too tightly around center.

**Fix**: Changed from cylindrical to full spherical spawn distribution:
```rust
let theta = rng.gen_range(0.0..std::f32::consts::TAU);
let phi = rng.gen_range(0.3..std::f32::consts::PI);  // Full sphere including bottom
let offset = Vec3::new(
    radius * phi.sin() * theta.cos(),
    radius * phi.cos(),
    radius * phi.sin() * theta.sin(),
);
```

### 4. Dot Sparkles Invisible

**Problem**: Dot sparkles weren't visible when spawned.

**Cause**: SmallDots texture is a multi-dot atlas - at small particle sizes, individual dots were sub-pixel.

**Fix**: Switched to GlowCircle texture which renders as single bright circle.

---

## Tower Integration

Replaced old `spawn_custom_shader_explosion` with the new combined WFX explosion in `pending_explosion_system`:

```rust
// Old (removed)
spawn_custom_shader_explosion(...);

// New
crate::wfx_spawn::spawn_combined_explosion(
    &mut commands,
    &mut meshes,
    &mut additive_materials,
    &mut smoke_materials,
    &mut smoke_only_materials,
    &asset_server,
    position,
    4.0,  // Scale
);
```

---

## Debug Keys

| Key | Effect |
|-----|--------|
| 1 | Center glow only |
| 2 | Explosion flames only |
| 3 | Smoke emitter only |
| 4 | Glow sparkles only |
| 5 | **Combined explosion (all 6 emitters)** |
| 6 | Dot sparkles (both types) |

---

## Code Cleanup

Removed 267 lines of unused legacy functions:
- `spawn_warfx_tower_explosion_test`
- `spawn_warfx_tower_explosion`
- `spawn_warfx_flame_burst`
- `spawn_warfx_smoke_column`

---

## Git History Cleanup

Removed accidentally committed files from history via interactive rebase:
- `resources/` folder (Unity reference materials, .cs, .shader, .mat files)
- Test shell scripts (run.sh, test_*.sh)

---

## Retrospective: The Full War FX Journey

This devlog marks the completion of War FX implementation, which spanned **5+ Claude Code threads** over several days. Here's a retrospective on the challenges, mistakes, and lessons learned.

### The Hardest Problems

#### 1. Bevy's Depth Prepass Limitation (Blocker)

**What we wanted**: Unity's soft particles fade based on scene depth - particles smoothly disappear when intersecting geometry.

**What we discovered**: Bevy 0.15's depth prepass is **not available for transparent/additive materials**. This is a fundamental architectural limitation, not a configuration issue.

**Why it matters**: Depth-based soft particles are impossible in Bevy for any material using `AlphaMode::Add` or `AlphaMode::Blend`. This required abandoning the Unity approach entirely.

**Workaround**: Texture-based soft edges using luminance-based alpha with `smoothstep()`:
```wgsl
let luminance = dot(tex.rgb, vec3<f32>(0.299, 0.587, 0.114));
let soft_alpha = smoothstep(0.0, 0.1, luminance);
```

#### 2. Blend Equation Mismatch (Multi-Session Debug)

**The mystery**: Rectangular quad borders were visible on smoke particles even with proper alpha.

**Root cause discovery**: Bevy's `AlphaMode::Multiply` uses a different blend equation than Unity's `Blend DstColor SrcAlpha`:

| Engine | Blend Equation |
|--------|----------------|
| **Bevy** | `dst * src.rgb + dst * (1 - src.a)` |
| **Unity** | `dst * src.rgb + dst * src.a` |

The difference is `(1 - src.a)` vs `src.a` - this completely broke Unity's `lerp(0.5, color, mask)` formula.

**Solution**: Override blend state via `Material::specialize()`:
```rust
target.blend = Some(BlendState {
    color: BlendComponent {
        src_factor: BlendFactor::Dst,
        dst_factor: BlendFactor::SrcAlpha,  // Match Unity exactly
        operation: BlendOperation::Add,
    },
    ...
});
```

**Time spent**: ~2 sessions debugging before finding Bevy's actual blend equation in `bevy_pbr::render::mesh.rs`.

##### Understanding the `lerp(0.5, color, mask)` Formula

The Unity shader uses this critical block:

```hlsl
// Texture sampling with mask
float mask = tex2D(_MainTex, i.uv).a * i.color.a;
fixed4 tex = tex2D(_MainTex, i.uv);
tex.rgb *= i.color.rgb * _TintColor.rgb;
tex.a = mask;
tex = lerp(fixed4(0.5,0.5,0.5,0.5), tex, mask);
```

**Line by line:**
1. `mask` = texture alpha (smoke shape) × particle alpha (fades over lifetime)
2. Sample texture color and tint it (orange for flames, brown for smoke)
3. `lerp(0.5, color, mask)` — interpolate between neutral gray and tinted color

**Why `lerp` is essential:**

The `lerp` creates a smooth gradient between invisible edges and visible centers:

| mask value | lerp result | effect on scene |
|------------|-------------|-----------------|
| 0 (edges) | `(0.5, 0.5, 0.5, 0.5)` | neutral (invisible) |
| 1 (center) | actual tinted color | visible smoke/flame |
| 0.5 | halfway blend | soft transition |

**Why 0.5 is "neutral"** with Unity's blend `dst * (src.rgb + src.a)`:
```
output (0.5, 0.5, 0.5, 0.5)  →  dst * (0.5 + 0.5) = dst * 1.0 = unchanged
```

**Visual comparison:**

```
Without lerp:                 With lerp:
┌─────────────────┐           ┌─────────────────┐
│▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓│           │                 │
│▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓│           │   ░░░░░░░░░░    │
│▓▓▓▓▓SMOKE▓▓▓▓▓▓▓│           │  ░▒▒▒▒▒▒▒▒▒░   │
│▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓│           │  ░▒▓SMOKE▓▒░   │
│▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓│           │  ░▒▒▒▒▒▒▒▒▒░   │
└─────────────────┘           │   ░░░░░░░░░░    │
  visible quad edge           │                 │
                              └─────────────────┘
                                soft falloff, no visible edge
```

Without lerp, the quad's rectangular boundary is visible. With lerp, the smoke texture's alpha gradient smoothly transitions to neutral gray at the edges, making the quad shape disappear entirely.

#### 3. Multiply Blend Math for Fire (Counterintuitive)

**The problem**: Orange flames were darkening the scene instead of brightening.

**The math**: With blend `dst * (rgb + alpha)`:
- Brightening requires `rgb + alpha > 1.0`
- Orange `(1.0, 0.522, 0.2)` only brightens red!
- Green (0.522) + alpha needs > 0.48 to brighten
- Blue (0.2) + alpha needs > 0.8 to brighten

**Solution**: Boost center colors toward warm white `(1.0, 0.95, 0.85)` for the "fusing" effect when particles overlap.

### Mistakes Made

1. **Assumed Bevy feature parity with Unity** - Wasted significant time trying to implement depth-based soft particles before discovering the architectural limitation.

2. **Didn't read Bevy source code early** - The blend equation difference was documented nowhere; had to dig into `mesh.rs` to find the actual implementation.

3. **Used wrong texture for dot sparkles** - SmallDots was a multi-dot atlas that became invisible at small sizes. Should have tested textures at target scale first.

4. **Forgot `* scale` in animation curves** - Simple oversight that caused sparkles to ignore the global scale parameter.

5. **Additive blend alpha misconception** - For additive blending, setting alpha=0 doesn't make things invisible if RGB>0. Had to multiply RGB by alpha too.

### What Went Well

1. **Detailed Unity prefab analysis** - Extracting exact particle counts, colors, curves from EXPLOSION_EMITTER_DETAILS.md saved massive guesswork.

2. **Incremental emitter development** - Building each emitter (glow, flames, smoke, sparkles) independently with debug keys (1-6) made testing much easier.

3. **Debug key system** - Being able to spawn individual emitters in isolation was invaluable for debugging.

4. **Color Over Lifetime curves** - Implementing Unity's piecewise linear interpolation faithfully resulted in accurate smoke-to-flame transitions.

### Technical Discoveries

| Discovery | Impact |
|-----------|--------|
| `Material::specialize()` can override blend state | Unlocked Unity-accurate blending |
| Bevy's depth prepass excludes transparent materials | Forced workaround approach |
| Additive blend needs RGB * alpha, not just alpha | Fixed glow hard-cut |
| `AsBindGroup` creates separate bindings, not packed structs | Fixed shader compilation |
| WGSL requires Vec4 alignment for uniforms | Fixed buffer mismatches |

### Statistics

- **Claude Code threads**: 5+
- **Sessions**: ~10
- **Devlogs written**: 8 (001-007 in wfx_explosion/, plus 012-013)
- **Custom shaders**: 2 (wfx_additive.wgsl, wfx_smoke_scroll.wgsl)
- **Custom materials**: 3 (AdditiveMaterial, SmokeScrollMaterial, SmokeOnlyMaterial)
- **Lines of Rust**: ~1,400 in wfx_spawn.rs
- **Particles per explosion**: ~214
- **Emitter types**: 6

### Key Files for Future Reference

- `devlogs/wfx_explosion/001-007*.md` - Detailed debugging sessions
- `devlogs/012_wfx-explosion-implementation-summary.md` - Technical summary
- `resources/wfx_explosivesmoke_big/EXPLOSION_EMITTER_DETAILS.md` - Unity prefab analysis (not committed)

---

## Reference

Effect inspired by [War FX](https://assetstore.unity.com/packages/vfx/particles/war-fx-5669) (free Unity asset, no license restrictions).
