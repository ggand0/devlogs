# GPU Fireball Gram-Schmidt Orientation Attempt - FAILED

**Date**: December 21, 2025
**Status**: ❌ Failed - Still shows mirroring
**Commits**:
- bevy_hanabi: (uncommitted changes to output.rs)
- bevy-mass-render: (uncommitted changes to particles.rs)

## Problem Recap

GPU fireballs using `AlongVelocity` orientation show horizontal texture mirroring when particles have opposite velocity directions. Previous attempts to fix with -90° rotation also caused mirroring due to cross product sign flips.

## Hypothesis

The issue was caused by the cross product `cross(dir, velocity)` changing sign when velocity direction changes. Combined with -90° rotation, this created odd-parity reflection (mirroring) instead of even-parity rotation.

**Solution attempted**: Replace cross product + rotation with direct Gram-Schmidt orthogonalization matching the CPU's proven VelocityAligned implementation.

## Implementation

### Changes Made

**1. bevy_hanabi/src/modifier/output.rs (lines 657-709)**

Replaced the AlongVelocity orientation code with Gram-Schmidt approach:

**Without rotation** (lines 690-707):
```wgsl
let velocity_dir = normalize(particle.velocity);
let to_camera = normalize(get_camera_position_effect_space() - position);

// Gram-Schmidt construction (matches CPU VelocityAligned)
axis_y = velocity_dir;
let forward_proj = dot(to_camera, axis_y);
let axis_z_unnorm = to_camera - forward_proj * axis_y;
let axis_z_len_sq = dot(axis_z_unnorm, axis_z_unnorm);

// Edge case: velocity parallel to camera
axis_z = select(
    normalize(vec3<f32>(0.0, 1.0, 0.0) - dot(vec3<f32>(0.0, 1.0, 0.0), axis_y) * axis_y),
    normalize(axis_z_unnorm),
    axis_z_len_sq > 0.001
);

axis_x = cross(axis_y, axis_z);
```

**With rotation** (lines 661-688): Same Gram-Schmidt base construction, then applies rotation in X-Y plane.

**Key features**:
- Y-axis aligned with velocity (matching CPU)
- Z-axis projected toward camera perpendicular to velocity (Gram-Schmidt)
- X-axis computed as cross product to complete orthonormal basis
- Edge case handling for velocity parallel to camera

**2. bevy-mass-render/src/particles.rs**

Removed the -90° rotation from fireball effect:
- Deleted line 949: `let fireball_rotation = writer_fireball.lit(std::f32::consts::FRAC_PI_2).expr();`
- Removed `.with_rotation(fireball_rotation)` from OrientModifier (line 972)
- Kept `.with_axis_rotation(fireball_axis_rotation)` for random spin feature

## Expected Outcome

Based on mathematical analysis:
- CPU uses Gram-Schmidt projection: `forward = (to_camera - up * dot(to_camera, up)).normalized`
- This creates axis_z (forward) independently without camera-dependent sign flips
- Should eliminate the mirroring issue caused by cross product sign coupling

**Expected**: GPU fireballs show consistent texture orientation across all velocity directions with no horizontal mirroring.

## Actual Result

❌ **STILL MIRRORING**

The Gram-Schmidt approach did not fix the horizontal texture mirroring issue. Particles with opposite velocity directions still show mirrored texture appearance.

## Analysis: Why It Failed

### Possible Explanations

**1. Axis Assignment May Not Be the Root Cause**

The mirroring issue might not be primarily about axis sign flips or orientation construction method. It could be:
- A texture UV coordinate problem
- A vertex shader vertex positioning issue
- A different handedness issue in how the quad is constructed
- The flipbook animation or texture sampling interacting with orientation

**2. GPU vs CPU Quad Construction Difference**

Even with matching axis orientation, there may be fundamental differences in how:
- Quad vertices are positioned using axes (`vpos = position + axis_x * vpos.x + axis_y * vpos.y + axis_z * vpos.z`)
- UV coordinates are mapped to vertices
- The billboard is facing the camera

**3. Gram-Schmidt Edge Case**

The WGSL `select()` fallback might behave differently than expected, or the threshold `0.001` might be triggering incorrectly, causing unexpected orientation switches.

**4. Coordinate System Handedness**

The cross product order `cross(axis_y, axis_z)` might produce a left-handed coordinate system when GPU expects right-handed (or vice versa), causing reflection even though axes are "correctly" oriented.

**5. The Problem May Be Elsewhere**

The mirroring might not be in the orientation code at all, but rather in:
- How the texture is sampled in the fragment shader
- How the flipbook animation frame selection works
- The UVScaleOverLifetimeModifier interaction
- The SimulationSpace::Local coordinate transformation

## Key Insights

1. **Gram-Schmidt doesn't help**: Changing the axis construction method from cross product to Gram-Schmidt projection has no effect on the mirroring
2. **Axis orientation alone isn't the issue**: Even with Y-axis along velocity (matching CPU), mirroring persists
3. **The real cause is deeper**: The problem is not simply "which axis points where" but something more fundamental about how the texture is mapped to the quad

## What We've Ruled Out

✅ Cross product sign flips (tried Gram-Schmidt instead)
✅ -90° rotation artifacts (removed rotation entirely)
✅ Camera-dependent axis sign changes (Gram-Schmidt eliminates this)
✅ Simple axis swapping (Y-axis now along velocity like CPU)

## What's Left to Investigate

**1. Vertex Shader Quad Construction**
- Examine `bevy_hanabi/src/render/vfx_render.wgsl` vertex positioning code
- Understand how `axis_x`, `axis_y`, `axis_z` are used to position quad vertices
- Check if UV coordinates are flipped or inverted based on axis signs

**2. Texture/UV Coordinate System**
- Investigate how texture UV coordinates map to quad vertices
- Check if there's a texture flip/mirror flag somewhere
- Examine FlipbookModifier UV calculations

**3. Handedness and Cross Product Order**
- Try `cross(axis_z, axis_y)` instead of `cross(axis_y, axis_z)`
- Explicitly check determinant of orientation matrix (should be +1 for right-handed)

**4. Compare with Other Orient Modes**
- Check how `ParallelCameraDepthPlane` constructs axes
- See if similar issues exist in other orientation modes

**5. CPU vs GPU Quad Geometry**
- Compare actual quad vertex positions in CPU vs GPU implementations
- Check if both use same vertex ordering (clockwise vs counter-clockwise)

## Files Changed

- `/home/gota/ggando/gamedev/bevy_hanabi/src/modifier/output.rs` (lines 657-709)
- `/home/gota/ggando/gamedev/bevy-mass-render/src/particles.rs` (lines 947-972)

## Current State

Code changes are uncommitted. The Gram-Schmidt implementation is in place but does not fix the issue.

## Next Steps

**Option A: Investigate quad vertex construction**
- Read `vfx_render.wgsl` to understand how axes transform vertex positions
- Check if vertex ordering or winding affects texture appearance

**Option B: Try axis handedness flip**
- Change `axis_x = cross(axis_y, axis_z)` to `axis_x = cross(axis_z, axis_y)`
- This inverts the X-axis, potentially fixing mirroring

**Option C: Examine texture sampling**
- Look at fragment shader texture sampling code
- Check if UV coordinates need flipping based on particle state

**Option D: Accept difference and adjust texture**
- GPU and CPU may fundamentally differ in quad construction
- Consider creating a horizontally flipped texture variant
- Or adjust UV coordinates in the effect setup

**Option E: Deep dive into bevy_hanabi rendering pipeline**
- Trace through the entire render path from orientation to final pixel
- Understand every transformation applied to the quad

## Conclusion

The Gram-Schmidt approach, while mathematically sound and matching the CPU implementation, **does not fix the mirroring issue**. This suggests the problem is not in axis construction but in how those axes are used to render the quad, or in the texture/UV coordinate system itself.

The issue is deeper than initially thought and requires investigation into the vertex shader, quad construction, and texture mapping rather than just the orientation axis calculation.
