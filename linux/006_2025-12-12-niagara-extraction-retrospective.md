# UE5 Niagara Extraction Retrospective: Lessons from 3 Passes

**Date**: 2025-12-12
**Purpose**: Document the challenges and lessons learned from extracting exact particle system details from UE5 Niagara T3D exports

## Overview

Porting a single UE5 Niagara particle system (NS_Explosion_Sand_5, 13 emitters) to Bevy required **3 full verification passes** over several days. Each pass uncovered a new category of subtle errors that made the particles look "wrong" in ways that were hard to diagnose.

This document captures what made extraction difficult and what to watch for in future ports.

## The Three Passes

### Pass 1: Initial Documentation
**Source**: MovieScene JSON exports, parameter dumps
**Result**: Particles looked fundamentally wrong - sizes off, colors brown instead of orange

**What was missing**:
- Curve data was incomplete
- UV scale effects not captured
- Velocity combination behavior unknown

### Pass 2: T3D Deep Dive
**Source**: Full T3D export (33,000+ lines)
**Result**: Much better, but subtle timing issues remained

**Key discoveries**:
- ShaderLUT is ground truth (not curve keyframes)
- UV scale curves (500→1) create zoom effects
- Multiple velocities combine at spawn, not sequentially
- Hex-encoded float parameters

### Pass 3: Fine-Tuning
**Source**: Side-by-side comparison, targeted T3D searches
**Result**: Finally matched reference

**Final fixes needed**:
- Alpha curves using smoothstep, not linear
- Flipbook animation modes (sequential vs random)
- Exact spawn positions (some at origin, some scattered)

## The Hardest Parts

### 1. ShaderLUT vs Curve Keys Mismatch

**The Problem**: Curve definitions look like one thing, but GPU samples something different.

Example that fooled me:
```
Curve=(Keys=((Time=0.800000,Value=1.000000),(Time=1.000000)))
```

I read this as "hold at 1.0 until 80%, then fade." Reality:
```
t=0.0: 1.00    t=0.2: 0.77    t=0.4: 0.56    t=0.6: 0.33    t=0.8: 0.14    t=1.0: 0.00
```

It was **fading from the start** due to cubic interpolation with tangents.

**Lesson**: Always extract and check ShaderLUT values. They're the actual GPU samples.

### 2. Smoothstep Everywhere (Not Linear)

**The Problem**: Assuming linear interpolation when Niagara uses cubic S-curves.

Dust alpha looked like it should be `3.0 * (1.0 - t)` but was actually:
```rust
let s = t * t * (3.0 - 2.0 * t);  // smoothstep
3.0 * (1.0 - s)
```

Smoke alpha looked like a simple parabola but was smoothstep bell curves on both sides.

**Symptom**: Particles "flash" or "pop" because opacity changes too abruptly at edges.

**Lesson**: When `InterpMode=RCIM_Cubic` appears, expect S-curve behavior.

### 3. Flipbook Animation Modes

**The Problem**: Assumed all flipbooks animate sequentially (frame 0→1→2→3→...).

Dust emitter had:
```
SubUV Animation Mode = NewEnumerator2
```

This means **random frame selection**, not sequential. Each particle picks one frame at spawn and keeps it.

**Symptom**: Dust looked too "animated" - frames cycling through created visual noise instead of static variety.

**Lesson**: Check `SubUV Animation Mode` enum:
- `NewEnumerator0` = Sequential (linear playback)
- `NewEnumerator2` = Random (fixed frame at spawn)

### 4. Hex-Encoded Parameters

**The Problem**: Parameters stored as byte arrays, not human-readable values.

```
VarData=(0,0,72,196,0,0,72,196,0,0,32,65)
```

This is a Vec3 in little-endian float format:
```python
import struct
struct.unpack('<fff', bytes([0,0,72,196,0,0,72,196,0,0,32,65]))
# Result: (-800.0, -800.0, 10.0)
```

**Lesson**: Write a decoder script early. I should have automated this from the start.

### 5. Multiple Velocity Sources

**The Problem**: Some emitters have multiple `AddVelocity` or `UniformRangedVector` modules. How do they combine?

Wisp emitter has two velocity ranges. I initially thought they applied as sequential forces or arc motion. Reality: they **add together at spawn time** to create a single combined velocity.

```rust
// WRONG - arc trajectory
let v1 = random_range(min1, max1);
let v2 = random_range(min2, max2);
// apply v1, then later v2...

// CORRECT - combined at spawn
let velocity = random_range(min1, max1) + random_range(min2, max2);
```

**Lesson**: Read the module execution order. Spawn modules run once; Update modules run every frame.

### 6. Local Space vs World Space

**The Problem**: `bLocalSpace=True` means velocity is relative to emitter orientation, not world.

Smoke velocity `(800, 800, 0)` in local space spreads toward/away from camera if emitter faces camera. In world space it would spread on ground plane.

**Lesson**: Always check `bLocalSpace` flag. It changes velocity interpretation completely.

### 7. Scale Starting at Zero

**The Problem**: Some particles are invisible at spawn and grow over lifetime.

Dust scale curve:
```
X: 0 → 3
Y: 0 → 2
```

If implementation doesn't handle zero scale properly, particles either:
- Don't appear at all (if culled when size=0)
- Appear at full size instantly (if defaulting to 1.0)

**Lesson**: Check scale curves for zero starting values.

### 8. Alpha > 1.0 (HDR Values)

**The Problem**: Alpha values can exceed 1.0 for HDR/bloom effects.

Dust alpha starts at **3.0**, not 1.0. Impact flash uses alpha **50000**.

If clamping alpha to 0-1 range, these effects don't work.

**Lesson**: Keep alpha unclamped in particle system. Let the shader/compositor handle it.

## Verification Workflow That Finally Worked

1. **Extract ShaderLUT for every curve** - Don't trust keyframe interpretation
2. **Decode all hex parameters** - Write a Python script for this
3. **Check animation mode for flipbooks** - Sequential vs Random changes everything
4. **Verify spawn position** - Some at origin, some in sphere, some scattered
5. **Test alpha at edges** - Linear vs smoothstep shows at t=0.1 and t=0.9
6. **Compare side-by-side** - Nothing beats visual comparison to reference

## Tools That Helped

### Grep Patterns
```bash
# Find emitter section
grep -n "EmitterName=\"dust\"" file.T3D

# Find ShaderLUT for a curve
grep -A 100 "dust.*Scale_Alpha" file.T3D | grep "ShaderLUT"

# Find animation mode
grep "SubUV Animation Mode" file.T3D
```

### Python Decoder
```python
import struct

def decode_float(hex_bytes):
    return struct.unpack('<f', bytes(hex_bytes))[0]

def decode_vec3(hex_bytes):
    return struct.unpack('<fff', bytes(hex_bytes))

# Example: VarData=(0,0,72,66)
decode_float([0,0,72,66])  # = 50.0
```

## Red Flags for Future Ports

| If you see... | Watch out for... |
|--------------|------------------|
| `InterpMode=RCIM_Cubic` | Smoothstep, not linear |
| `NewEnumerator2` in SubUV | Random frame selection |
| Multiple velocity modules | Combination at spawn |
| `bLocalSpace=True` | Velocity relative to emitter |
| Scale curve starting at 0 | Invisible particles at spawn |
| Alpha > 1.0 | HDR effects, don't clamp |
| Curve keys != LUT values | Use ShaderLUT as ground truth |

## Time Investment

| Phase | Time | What |
|-------|------|------|
| Pass 1 | ~4 hours | Initial doc from JSON, basic implementation |
| Pass 2 | ~6 hours | T3D deep dive, major corrections |
| Pass 3 | ~3 hours | Fine-tuning alpha curves, flipbook modes |
| **Total** | **~13 hours** | For 13 emitters |

Most time was spent on the "it looks almost right but something's off" debugging in Pass 3.

## Conclusion

The hardest part of Niagara extraction isn't finding the data - it's in the T3D file. The hard part is **knowing which data matters** and **how Niagara interprets it**.

Key takeaways:
1. **ShaderLUT is truth** - Curve keys are just hints
2. **Smoothstep is default** - Assume cubic interpolation
3. **Check enum values** - Animation modes, coordinate spaces
4. **Decode hex early** - Automate parameter extraction
5. **Visual comparison is essential** - Numbers can match but still look wrong

For the next Niagara port, I'd spend more time upfront building extraction tools and less time manually reading T3D files.
