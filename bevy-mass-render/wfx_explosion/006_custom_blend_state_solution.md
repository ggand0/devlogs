# Devlog 003: Custom Blend State - Rectangular Border Fix

**Date**: 2025-11-27
**Branch**: `feat/warfx-explosion`
**Commit**: `9df2f09`

## Problem Solved

Rectangular quad borders were visible on smoke particles. The edges showed clear quad shapes instead of blending smoothly into the scene.

## Root Cause

Bevy's `AlphaMode::Multiply` uses a different blend equation than Unity's `Blend DstColor SrcAlpha`:

| Engine | Blend Equation |
|--------|----------------|
| **Bevy** | `dst * src.rgb + dst * (1 - src.a)` |
| **Unity** | `dst * src.rgb + dst * src.a = dst * (src.rgb + src.a)` |

The critical difference is `(1 - src.a)` vs `src.a` for the destination factor.

### Why This Matters

Unity's `lerp(0.5, color, mask)` formula outputs:
- At edges (mask=0): `rgb=0.5, alpha=0.5` → blend factor = `0.5 + 0.5 = 1.0` (neutral!)
- At center (mask=1): `rgb=color, alpha=1.0` → blend factor = `color + 1.0`

With Bevy's default multiply blend, the math doesn't work out to neutral edges.

## Solution

Override the blend state via `Material::specialize()`:

```rust
fn specialize(
    _pipeline: &MaterialPipeline<Self>,
    descriptor: &mut RenderPipelineDescriptor,
    _layout: &MeshVertexBufferLayoutRef,
    _key: MaterialPipelineKey<Self>,
) -> Result<(), SpecializedMeshPipelineError> {
    if let Some(ref mut fragment) = descriptor.fragment {
        for target in fragment.targets.iter_mut().flatten() {
            target.blend = Some(BlendState {
                color: BlendComponent {
                    src_factor: BlendFactor::Dst,
                    dst_factor: BlendFactor::SrcAlpha,
                    operation: BlendOperation::Add,
                },
                alpha: BlendComponent {
                    src_factor: BlendFactor::One,
                    dst_factor: BlendFactor::OneMinusSrcAlpha,
                    operation: BlendOperation::Add,
                },
            });
        }
    }
    Ok(())
}
```

## Shader Changes

Removed alpha inversion - use raw texture alpha like Unity:

```wgsl
// Sample texture alpha directly (no inversion needed)
let tex_alpha = textureSample(smoke_texture, smoke_sampler, uv).a;

// Unity formula
let mask = tex_alpha * particle_alpha;

// Unity's lerp(0.5, color, mask) for BOTH rgb AND alpha
let final_rgb = mix(vec3<f32>(0.5), lifetime_color, mask);
let final_alpha = mix(0.5, 1.0, mask);
```

## Result

- No more rectangular borders
- Bright core with soft particle effect
- Particles blend smoothly into scene

## Files Modified

- `src/wfx_materials.rs` - Added `specialize()` with custom blend state
- `assets/shaders/wfx_smoke_scroll.wgsl` - Removed alpha inversion

## Remaining Work

- Smoke transition (dark smoke at end of lifetime) needs verification
- UV scrolling not yet implemented
- Integration with full explosion effect
