# Devlog: War FX Explosion Glow Effects Implementation

**Date:** November 25, 2025
**Branch:** `feat/warfx-explosion`
**Focus:** Implementing Unity War FX asset pack glow effects with soft particle techniques

---

## Overview

This session focused on implementing high-quality explosion glow effects using the War FX asset pack from Unity. The goal was to create realistic, soft-edged particle effects that blend naturally with the scene - similar to Unity's soft particles feature which uses depth-based fade.

## Context

The project is a Bevy 0.15-based RTS game with 10,000+ unit battles. Previous explosion implementations used flipbook sprite sheets, but we wanted to add more sophisticated visual effects using professional VFX assets. The War FX asset pack provides high-quality textures and materials designed for Unity, which needed to be adapted to Bevy's rendering pipeline.

---

## Initial Implementation: Depth-Based Soft Particles

### The Goal

Soft particles create smooth, natural-looking borders by fading particles to transparent when they intersect with scene geometry. This prevents the harsh "circular cutout" effect that occurs when billboard particles clip through objects.

**Unity Implementation:**
- Uses depth prepass to capture scene depth
- Compares particle depth to scene depth in fragment shader
- Fades alpha based on distance: `alpha *= saturate(InvFade * (sceneZ - partZ))`
- Parameter: `InvFade` (default: 1.0) controls fade distance

### Bevy Implementation Attempt

#### Step 1: Material Structure

Created `AdditiveMaterial` with soft particle support:

```rust
#[derive(Asset, TypePath, AsBindGroup, Debug, Clone)]
pub struct AdditiveMaterial {
    #[uniform(0)]
    pub tint_color: Vec4,

    #[uniform(1)]
    pub soft_particles_fade: Vec4,  // x = inv_fade

    #[texture(2)]
    #[sampler(3)]
    pub particle_texture: Handle<Image>,
}
```

**Shader bindings:**
- `@group(2) @binding(0)` - tint_color
- `@group(2) @binding(1)` - soft_particles_fade
- `@group(2) @binding(2)` - particle_texture
- `@group(2) @binding(3)` - particle_sampler

#### Step 2: Camera Depth Prepass

Added depth prepass to camera setup:

```rust
use bevy::core_pipeline::prepass::DepthPrepass;

commands.spawn((
    Camera3dBundle { /* ... */ },
    RtsCamera { /* ... */ },
    DepthPrepass,  // Enable depth prepass
));
```

#### Step 3: Shader Depth Access Attempts

**Attempt 1: Manual depth texture binding**
```wgsl
@group(0) @binding(4)
var depth_texture: texture_depth_2d;
```
❌ Error: Binding not available in pipeline layout

**Attempt 2: Different binding numbers**
Tried various binding indices
❌ Error: View dimension mismatch (D2 vs D2Array)

**Attempt 3: Bevy's prepass utilities**
```wgsl
#import bevy_pbr::prepass_utils
let scene_depth = prepass_utils::prepass_depth(in.position, 0u);
```
❌ Result: Particles became invisible

**Attempt 4: Debug visualization**
Added color-coded depth visualization
❌ Result: Entire screen became colored, particles invisible

### The Fundamental Limitation

After extensive investigation and documentation review, discovered:

> **Bevy's depth prepass textures are not generated for any material using alpha blending.**

**Key insight from [src/wfx_materials.rs:67](src/wfx_materials.rs#L67):**
```rust
fn alpha_mode(&self) -> AlphaMode {
    AlphaMode::Add  // ONE + ONE additive blending
}
```

**The problem:**
- `AlphaMode::Add` is a transparent blending mode
- Bevy 0.15's depth prepass **explicitly excludes transparent materials**
- Depth-based soft particles are fundamentally incompatible with additive materials
- This is a core architectural limitation, not a configuration issue

**Why this limitation exists:**
- Transparent objects don't write to depth buffer by default
- Depth buffer is written during opaque pass
- Additive materials render in transparent pass (after depth)
- Chicken-and-egg problem: need depth to fade, but can't write depth

### Documentation

Added comprehensive comments to shader explaining the limitation:

```wgsl
// NOTE: Soft particles (depth-based fade) are not currently supported.
// Bevy's depth prepass is not available for transparent/additive materials (AlphaMode::Add).
// This is a fundamental limitation of Bevy 0.15's rendering architecture.
// To implement soft particles would require a custom render pipeline or post-processing effect.
```

---

## Alternative Solution: Texture-Based Soft Edges

### The Approach

Since depth-based fade wasn't possible, focused on making the particle textures themselves have softer edges through texture manipulation and shader techniques.

### Shader Implementation

**Key technique: Luminance-based alpha with smoothstep**

```wgsl
// Unity "A8" textures use RGB luminance as alpha mask
let luminance = dot(tex.rgb, vec3<f32>(0.299, 0.587, 0.114));

// Discard only completely black pixels
if (luminance < 0.001) {
    discard;
}

// Apply soft falloff for smooth blending
let soft_alpha = smoothstep(0.0, 0.1, luminance);

// Final color with soft edges
let final_color = vec4<f32>(
    tex.rgb * tint.rgb * brightness,
    soft_alpha * tint.a
);
```

**Parameters evolved:**
1. **Discard threshold:** `0.05` → `0.001`
   - Preserves more edge detail
   - Prevents premature cutoff of fade gradient

2. **Smoothstep range:** `smoothstep(0.0, 0.1, luminance)`
   - Creates gradual alpha transition from 0 to 1
   - Maps luminance 0.0-0.1 to alpha 0.0-1.0 with Hermite interpolation
   - Much softer than linear fade

3. **Brightness multiplier:** `4.0`
   - Matches Unity shader's `2.0 * 2.0`
   - Compensates for additive blending

### Visual Results

**Before:**
- Hard circular borders on particles
- Visible "cookie cutter" shape
- Obvious when particles overlap geometry

**After:**
- Smooth gradual fade at edges
- Natural-looking particle dissipation
- Much closer to professional VFX look
- User feedback: "Cool! I feel this is much closer to the original effect"

### Remaining Gap

User noted: "I feel like the original glow has even softer edge, as if it applied gaussian noise or something."

This is likely due to:
1. Unity's depth-based soft particles (not possible in Bevy)
2. Additional blur/noise passes in Unity (performance cost)
3. Multiple overlapping particle layers (could add)

**Decision:** Current implementation is "good progress" - acceptable quality for scope

---

## Technical Deep Dive

### A8 Texture Format

Unity's War FX uses "A8" texture format:
- RGB channels contain luminance (grayscale)
- No separate alpha channel
- Alpha is **derived** from RGB brightness
- Shader calculates: `alpha = dot(rgb, luminance_weights)`

**Why this matters:**
- Texture artist controls fade through brightness
- More intuitive than separate alpha channel
- Compresses better (single grayscale gradient)

### Smoothstep Function

The Hermite interpolation function:
```
smoothstep(edge0, edge1, x) = t * t * (3.0 - 2.0 * t)
where t = clamp((x - edge0) / (edge1 - edge0), 0.0, 1.0)
```

**For our case:** `smoothstep(0.0, 0.1, luminance)`
- **Input range:** 0.0 to 0.1 luminance
- **Output:** 0.0 to 1.0 alpha (smooth S-curve)
- **Effect:** Much softer transition than `alpha = luminance`

**Comparison:**
- Linear: `alpha = luminance` → straight diagonal
- Smoothstep: `alpha = smoothstep(...)` → ease-in/ease-out S-curve

### AsBindGroup Automatic Bindings

Bevy's `AsBindGroup` derive creates shader bindings automatically:

```rust
#[derive(AsBindGroup)]
pub struct AdditiveMaterial {
    #[uniform(0)]       // → @group(2) @binding(0)
    pub tint_color: Vec4,

    #[uniform(1)]       // → @group(2) @binding(1)
    pub soft_particles_fade: Vec4,

    #[texture(2)]       // → @group(2) @binding(2)
    #[sampler(3)]       // → @group(2) @binding(3)
    pub particle_texture: Handle<Image>,
}
```

**Critical lesson:** Each `#[uniform]` gets its own binding, NOT packed into a struct.

**Initial mistake:**
```wgsl
// WRONG - tried to access as struct
struct AdditiveMaterial {
    tint_color: vec4<f32>,
    soft_particles_fade: vec4<f32>,
}
@group(2) @binding(0)
var<uniform> material: AdditiveMaterial;
```

**Correct approach:**
```wgsl
// RIGHT - separate bindings
@group(2) @binding(0)
var<uniform> tint_color: vec4<f32>;
@group(2) @binding(1)
var<uniform> soft_particles_fade: vec4<f32>;
```

### WGSL Uniform Alignment

WGSL requires Vec4 alignment for uniform structs:

**Cannot do:**
```rust
pub struct AdditiveMaterial {
    pub tint_color: Vec4,
    pub soft_particles_inv_fade: f32,  // ❌ Causes alignment issues
}
```

**Must use:**
```rust
pub struct AdditiveMaterial {
    pub tint_color: Vec4,
    pub soft_particles_fade: Vec4,  // ✅ x = inv_fade, yzw = padding
}
```

**Accessing in shader:**
```wgsl
let inv_fade = soft_particles_fade.x;  // Only use x component
```

---

## Implementation Details

### Files Modified

1. **[assets/shaders/wfx_additive.wgsl](assets/shaders/wfx_additive.wgsl)** (55 lines)
   - Additive particle shader with soft edges
   - Luminance-based alpha calculation
   - Smoothstep for edge falloff
   - Documented depth prepass limitation

2. **[src/wfx_materials.rs](src/wfx_materials.rs)** (88 lines)
   - `AdditiveMaterial` definition
   - `AlphaMode::Add` for bright glow
   - `soft_particles_fade` uniform (reserved for future)
   - `SmokeScrollMaterial` for animated smoke

3. **[src/wfx_spawn.rs](src/wfx_spawn.rs)** (functions: `spawn_warfx_center_glow`, `spawn_warfx_flame_burst`)
   - Initialize materials with default InvFade = 1.0
   - Spawn glow billboards at explosion center
   - Scale and lifetime parameters

4. **[src/setup.rs](src/setup.rs)**
   - Added `DepthPrepass` to camera (kept for future)
   - Will remain useful if/when depth access becomes available

### Shader Code (Final Version)

```wgsl
#import bevy_pbr::forward_io::VertexOutput

@group(2) @binding(0)
var<uniform> tint_color: vec4<f32>;
@group(2) @binding(1)
var<uniform> soft_particles_fade: vec4<f32>;
@group(2) @binding(2)
var particle_texture: texture_2d<f32>;
@group(2) @binding(3)
var particle_sampler: sampler;

@fragment
fn fragment(in: VertexOutput) -> @location(0) vec4<f32> {
    var tex = textureSample(particle_texture, particle_sampler, in.uv);
    let tint = tint_color;

    // Calculate luminance for alpha
    let luminance = dot(tex.rgb, vec3<f32>(0.299, 0.587, 0.114));

    // Very low threshold to preserve soft falloff
    if (luminance < 0.001) {
        discard;
    }

    // Smooth edge falloff
    let soft_alpha = smoothstep(0.0, 0.1, luminance);

    // 4x brightness (Unity shader: 2.0 * 2.0)
    let brightness = 4.0;
    let final_color = vec4<f32>(
        tex.rgb * tint.rgb * brightness,
        soft_alpha * tint.a
    );

    return final_color;
}
```

### Material Spawning

```rust
let glow_material = additive_materials.add(AdditiveMaterial {
    tint_color: Vec4::new(1.0, 1.0, 1.0, 1.0),
    soft_particles_fade: Vec4::new(1.0, 0.0, 0.0, 0.0),  // InvFade = 1.0
    particle_texture: glow_texture.clone(),
});

commands.spawn((
    MaterialMeshBundle {
        mesh: quad_mesh,
        material: glow_material,
        transform,
        ..default()
    },
    WarFxGlow { /* ... */ },
));
```

---

## Troubleshooting Journey

### Issue 1: Shader Compilation Errors

**Problem:** Multiple binding/struct errors during shader development

**Errors encountered:**
```
error: invalid field accessor `clip_position`
error: Shader global ResourceBinding { group: 2, binding: 0 } not available
error: Buffer structure size mismatch
```

**Root causes:**
1. Used `in.clip_position.z` instead of `in.position.z`
2. Tried struct-based uniform instead of separate bindings
3. Used `f32` instead of `Vec4` for alignment

**Solution:** Systematic debugging with shader compiler output, documentation review

### Issue 2: Invisible Particles

**Problem:** Particles rendered but weren't visible on screen

**Cause:** Debug depth visualization code replaced texture color entirely

**Solution:** Removed debug code, focused on texture-based approach

### Issue 3: Hard Edges

**Problem:** Circular borders visible at particle edges

**Initial parameters:**
- Discard threshold: `0.05`
- No smoothstep

**Solution:**
- Reduced threshold: `0.05` → `0.001`
- Added smoothstep: `smoothstep(0.0, 0.1, luminance)`

### Issue 4: Understanding Bevy's Depth System

**Problem:** Confusion about why depth access wasn't working

**Investigation steps:**
1. Tried various binding numbers
2. Checked prepass utilities documentation
3. Reviewed Bevy source code
4. Found limitation in Bevy book documentation

**Resolution:** Depth prepass fundamentally incompatible with transparent materials

---

## Performance Characteristics

### Shader Complexity

**Simple and fast:**
- Single texture sample
- One dot product (luminance)
- One smoothstep (3 operations)
- One multiplication cascade

**No expensive operations:**
- No loops
- No branching (except discard)
- No noise functions
- No multi-pass

### Material Efficiency

**Additive blending:**
- ONE source + ONE dest = bright accumulation
- GPU-native operation
- No special handling needed

**Billboard rendering:**
- Single quad per particle
- 2 triangles, 4 vertices
- Minimal geometry overhead

### Scalability

**Current implementation:**
- Multiple glow particles per explosion
- Each explosion spawns 1-3 glows
- Hundreds of explosions possible simultaneously
- Maintained frame rate with mass battles

---

## Lessons Learned

### 1. Platform Limitations Matter

Don't assume features from one engine (Unity) are possible in another (Bevy). Depth-based soft particles work in Unity but not in Bevy 0.15 for transparent materials.

**Takeaway:** Research engine capabilities early, before committing to implementation approach.

### 2. Texture-Based Techniques Are Powerful

Even without depth access, sophisticated visual effects are possible through:
- Careful texture authoring
- Smart shader math (smoothstep)
- Understanding of alpha blending

**Takeaway:** Multiple paths to same visual goal; be flexible.

### 3. Uniform Binding Systems Vary

Unity's material system vs Bevy's `AsBindGroup` work completely differently:
- Unity: Nested structs, automatic packing
- Bevy: Flat bindings, explicit indices

**Takeaway:** Study target engine's shader binding conventions carefully.

### 4. User Feedback Drives Quality

Initial implementation had hard edges → user complained → improved with smoothstep.

**Takeaway:** Iterate based on visual feedback, not just technical correctness.

### 5. Document Limitations Prominently

Clear inline documentation about depth prepass limitation prevents future confusion:
```rust
// NOTE: Soft particles (depth-based fade) are not currently supported.
```

**Takeaway:** Future maintainers (including yourself) will thank you.

---

## Future Improvements

### Possible Enhancements

1. **Custom Render Pipeline**
   - Implement depth access for transparent materials
   - Requires significant Bevy rendering knowledge
   - High complexity, uncertain timeline

2. **Gaussian Blur Post-Process**
   - Apply blur to glow particles specifically
   - Could achieve "softer" look user mentioned
   - Performance cost: additional render pass

3. **Multi-Layer Particles**
   - Spawn multiple overlapping glows
   - Different scales, opacities
   - Creates depth through parallax

4. **Noise Texture Overlay**
   - Add subtle noise to break up uniformity
   - Modulate alpha or brightness
   - Cheap to implement

5. **Animated Noise**
   - UV distortion based on time
   - Creates "flickering" organic feel
   - Requires additional shader math

### What Won't Be Pursued

1. **Depth-based soft particles**
   - Fundamentally blocked by Bevy architecture
   - Would require engine contribution or custom fork
   - Not worth effort for current scope

2. **Compute shader particles**
   - Overkill for current needs
   - Complexity not justified
   - Current billboard approach sufficient

---

## Conclusion

Successfully implemented War FX explosion glow effects in Bevy despite discovering that Unity's depth-based soft particle technique is impossible in Bevy 0.15 for transparent materials. Adapted by using texture-based soft edges with smoothstep interpolation, achieving visually acceptable results that the user confirmed are "much closer to the original effect."

**Key outcomes:**
- ✅ Soft-edged glow particles without hard borders
- ✅ Additive blending for bright accumulation
- ✅ Performance-friendly shader implementation
- ✅ Documented limitations for future reference
- ⚠️ Slightly less soft than Unity version (acceptable trade-off)

**Technical achievements:**
- Custom material with WGSL shader
- Proper AsBindGroup usage
- Luminance-based alpha calculation
- Smoothstep edge falloff
- Billboard rendering integration

**Knowledge gained:**
- Bevy's depth prepass architecture
- WGSL uniform alignment requirements
- AsBindGroup binding generation
- Unity-to-Bevy material translation
- Texture-based VFX techniques

This implementation provides a solid foundation for explosion visual effects and demonstrates that high-quality VFX is achievable in Bevy even when direct engine feature parity with Unity isn't possible.

---

**Next Steps:**

Based on user's draft note: "Thanks, this works as a baseline of explosion effects. I'd like to explore more on how to improve this in the following order: 1. particle"

Likely next task will involve implementing GPU particle systems for debris, sparks, and smoke to complement the glow sprite sheets.

---

**Files Changed:**
- `assets/shaders/wfx_additive.wgsl` (new)
- `src/wfx_materials.rs` (new)
- `src/wfx_spawn.rs` (modified)
- `src/setup.rs` (modified - added DepthPrepass)

**Commit:** "Implement War FX glow effects with texture-based soft edges"
