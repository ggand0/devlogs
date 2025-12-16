# Devlog 004: Color Tuning and Bright Fusing Cores

**Date**: 2025-11-28
**Branch**: `feat/warfx-explosion`
**Commits**: `c453599`, `3b729f6`

## Problem

After fixing the rectangular borders (devlog 003), the smoke effect was working but:
1. Core color was completely white instead of orange/red
2. The bright "fusing" effect when particles overlap was weak

## Root Cause Analysis

### Issue 1: White Cores

The previous shader used:
```wgsl
var flame_color = mix(tint_color, vec3<f32>(1.0), 0.8);
flame_color = max(flame_color, vec3<f32>(0.85));
```

This pushed the flame color 80% toward white, making cores appear completely white instead of the Unity orange.

### Issue 2: Weak Fusing Effect

For the multiply blend `dst * (rgb + alpha)`:
- **Brightening** requires `rgb + alpha > 1.0`
- Orange `(1.0, 0.522, 0.2)` only brightens the red channel
- Green (0.522) needs alpha > 0.48 to exceed 1.0
- Blue (0.2) needs alpha > 0.8 to exceed 1.0

So pure orange actually **darkens** the green and blue channels!

## Solution

### Unity's Exact Colors

From `EXPLOSION_EMITTER_DETAILS.md`, Unity's Start Color Gradient:

| Position | RGB | Visual |
|----------|-----|--------|
| 0% | (1.0, 0.922, 0.827) | Cream |
| 9% | (1.0, 0.522, 0.2) | **Bright orange** (flame) |
| 42% | (0.663, 0.235, 0.184) | **Dark brown** (smoke) |
| 100% | (0.996, 0.741, 0.498) | Tan |

### Color Over Lifetime Curve

Unity also applies a Color Over Lifetime multiplier:

| Time | Multiplier |
|------|------------|
| 0% | 1.0 (white) |
| 20% | 0.694 |
| 41% | 0.404 (darkest) |
| 100% | 0.596 |

Implemented as piecewise linear interpolation:
```wgsl
let lifetime_t = 1.0 - particle_alpha;  // 0→1 over lifetime
let lifetime_mult = select(
    mix(1.0, 0.694, lifetime_t / 0.2),
    select(
        mix(0.694, 0.404, (lifetime_t - 0.2) / 0.21),
        mix(0.404, 0.596, (lifetime_t - 0.41) / 0.59),
        lifetime_t > 0.41
    ),
    lifetime_t > 0.2
);
```

### Bright Fusing Cores

To get the bright fusing effect while keeping orange tint, we blend toward warm white at particle centers:

```wgsl
// center_boost: strongest at texture center + start of life
let center_boost = tex_alpha * particle_alpha;

// Blend toward warm white (1.0, 0.95, 0.85) at centers
let bright_flame = mix(darkened_flame, vec3<f32>(1.0, 0.95, 0.85), center_boost * 0.7);
```

This ensures:
- At center (high tex_alpha) + spawn (high particle_alpha): strong boost toward white → fusing cores
- At edges or later in life: keeps orange/brown colors for smoke effect

### Spatial Color Distribution

Using texture alpha for spatial blending:
```wgsl
let spatial_blend = tex_alpha * (0.5 + 0.5 * particle_alpha);
let pixel_color = mix(darkened_smoke, bright_flame, spatial_blend);
```

- **Center** (high tex_alpha): bright flame color
- **Edges** (low tex_alpha): dark smoke color
- **Over lifetime**: center stays bright longer, edges darken first

## Final Shader Flow

1. Sample texture alpha (high at center, low at edges)
2. Calculate mask: `tex_alpha * particle_alpha`
3. Apply Color Over Lifetime darkening to base colors
4. Boost center toward warm white for fusing effect
5. Spatial blend: center=flame, edges=smoke
6. Apply Unity's `lerp(0.5, color, mask)` formula
7. Tie alpha to color for proper blend math

## Result

- Orange/red flame cores instead of pure white
- Bright fusing effect when particles overlap
- Dark smoke at edges and toward end of lifetime
- Smooth transition from flame to smoke over particle lifetime

## Files Modified

- `assets/shaders/wfx_smoke_scroll.wgsl` - Color tuning and fusing cores
