# UE5 Niagara Particle System Analysis via T3D Export

**Date**: 2025-12-12
**Purpose**: Document the process of extracting detailed Niagara particle system data from UE5 T3D exports for Bevy implementation

## Overview

Successfully extracted and documented all 13 emitters from the UE5 `NS_Explosion_Sand_5` Niagara system by analyzing the raw T3D export file (33,000+ lines). This approach revealed critical curve data and parameters that weren't visible in simpler JSON exports.

## The Problem with Initial Approach

Initial documentation was created from:
- MovieScene JSON exports
- Simplified parameter dumps

**What was missing:**
- Actual curve keyframe data
- ShaderLUT values (pre-computed curve samples)
- UV scale curves
- Correct velocity combination behavior

This led to incorrect implementations where particles looked "too large" or "brown" when they shouldn't be.

## T3D File Structure

The T3D export contains the complete Niagara system serialized as Unreal's text-based format:

```
Begin Object Class=/Script/Niagara.NiagaraSystem Name="NS_Explosion_Sand_5"
   ...
   EmitterHandles(0)=(Instance=..., Name="smoke")
   EmitterHandles(1)=(Instance=..., Name="main")
   ...
End Object
```

### Key Data Locations

| Data Type | Location in T3D |
|-----------|-----------------|
| Emitter names | `EmitterName="xxx"` |
| Curve data | `NiagaraDataInterfaceCurve` objects |
| Float parameters | `RapidIterationParameters` with hex-encoded floats |
| Velocity ranges | `UniformRangedVector.Min/Max` in hex |
| Material refs | `RendererProperties` sections |

## Critical Discovery: ShaderLUT vs Curve Keys

The curve definition and actual behavior can differ significantly:

```
Curve=(Keys=((InterpMode=RCIM_Cubic,Time=0.800000,Value=1.000000,ArriveTangent=-1.052632),(Time=1.000000)))
```

This **looks like** "hold at 1.0 until t=0.8, then fade". But the actual ShaderLUT reveals:

```
ShaderLUT(0)=1.000000   // t=0.0
ShaderLUT(26)=0.500000  // t=0.5 - already at 50%!
ShaderLUT(52)=0.000000  // t=1.0
```

**The LUT is the ground truth** - it's what the GPU actually samples.

## Emitters Documented

| Emitter | Type | Key Features |
|---------|------|--------------|
| main | Fireball | VelocityAligned, UV scale 500→1, 81 frames |
| main001 | Fireball | Same as main, different texture (64 frames) |
| smoke | Smoke cloud | CurlNoise force, 8×8 flipbook |
| dust | Ground dust | 4×1 flipbook, Vector2D scale |
| wisp | Smoke tendrils | Two velocities combined at spawn |
| dirt | Debris clumps | Random UV rotation, collision |
| dirt001 | Secondary dirt | Smaller, faster |
| spark | Emissive sparks | HDR color (R=50), velocity stretch |
| spark_l | Large sparks | "Shooting star" scale curve |
| impact | Flash | HDR 50000, 0.1s lifetime |
| parts | Mesh debris | 3 mesh variants, physics, rolling |
| refr_sprite | Distortion | Normal map refraction |
| refr_mesh | Distortion | Sphere mesh refraction |

## Key Findings

### 1. Alpha Curves Often Fade From Start

Documentation initially said "holds at 1.0 for 80%". Reality from LUT:
```
t=0.0: 1.00    t=0.2: 0.77    t=0.4: 0.56    t=0.6: 0.33    t=0.8: 0.14    t=1.0: 0.00
```

### 2. UV Scale Creates Zoom Effect

The `main` emitters have a 500→1 UV scale curve that creates a "zoom out" effect:
- Start: UV scaled 500×, showing only center pixel of texture
- End: UV scaled 1×, showing full texture

This is a major visual effect that wasn't documented initially.

### 3. Multiple Velocities Combine at Spawn

The `wisp` emitter has two `UniformRangedVector` components that are **added together at spawn**, not applied sequentially:

```rust
// WRONG - applying as arc motion
velocity = vec1; // then later apply vec2

// CORRECT - combine at spawn
velocity = vec1 + vec2; // single combined velocity
```

This creates scattered drift patterns, not arc trajectories.

### 4. Hex-Encoded Float Parameters

Parameters are stored as little-endian hex bytes:
```
VarData=(0,0,72,66)  // bytes for float
```

Decode with Python:
```python
import struct
struct.unpack('<f', bytes([0,0,72,66]))[0]  # = 50.0
```

## Tips for Extracting UE5 Niagara Data

### 1. Use T3D Export, Not JSON

- T3D contains complete serialized data
- JSON exports often omit curve LUTs
- T3D is searchable with grep/regex

### 2. Search Patterns

```bash
# Find emitter by name
grep -n "EmitterName=\"smoke\"" file.T3D

# Find curve data for emitter
grep -A 20 "SourceEmitterName=\"main\"" file.T3D | grep -i curve

# Find specific curve object
grep -A 50 "NiagaraDataInterfaceCurve_6" file.T3D
```

### 3. Trust ShaderLUT Over Curve Keys

The `ShaderLUT(n)=value` entries are pre-computed samples. Use these to understand actual behavior:
- LUT typically has 256 entries (0-255)
- Index maps to normalized time: `t = index / 255`

### 4. Check All Curve Types

| Curve Type | Purpose |
|------------|---------|
| `NiagaraDataInterfaceCurve` | Float values (scale, alpha, UV) |
| `NiagaraDataInterfaceColorCurve` | RGBA color over time |
| `NiagaraDataInterfaceVector2DCurve` | 2D values (sprite size X/Y) |

### 5. Cross-Reference Material Graphs

Material graphs in T3D show how particle data is used:
- `ParticleColor` → base color
- `DynamicMaterialParameter[n]` → custom params
- `DepthFade` → soft particle blending

### 6. Watch for Combined Parameters

Some parameters combine multiple systems:
- Lifetime = base × multiplier
- Velocity = vec1 + vec2 (at spawn)
- Scale = base_size × scale_curve × uv_scale

## Corrected Documentation

Updated emitter docs with:
- Actual LUT samples for curves
- UV scale curves where present
- Correct velocity combination behavior
- Lifetime multipliers
- Notes on visual discrepancies (brown color causes)

## Lessons Learned

1. **Never trust curve keys alone** - always check ShaderLUT
2. **Multiple velocity sources combine** - read module execution order
3. **UV effects are often hidden** - look for `NiagaraDataInterfaceCurve` without obvious names
4. **Test against reference** - compare implementation to UE5 preview constantly
5. **Document LUT samples** - easier to implement than interpreting curve tangents

## File References

- T3D Export: `resources/ue5_explosion/NS_Explosion_Sand_5.T3D` (33,000+ lines)
- JSON Summary: `resources/ue5_explosion/NS_Explosion_Sand_5_detailed.json`
- Emitter Docs: `resources/ue5_explosion/emitters/*.md` (13 files)
