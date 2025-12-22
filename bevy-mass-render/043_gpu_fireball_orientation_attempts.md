# GPU Fireball Orientation Fix Attempts - Session 2

**Date**: December 21-22, 2025
**Status**: In Progress (BLOCKED)
**Branch**: feat/gpu-particle-perf

## Problem Summary

GPU fireballs using `AlongVelocity` orientation show **visual mirroring** that makes particles appear to travel in the wrong direction:

- **CPU behavior**: Particles spawn on a circle and fly **outward** from the explosion center
- **GPU behavior (mirrored)**: Particles appear to travel **inward then across** to the other side

This is NOT just a texture flip - the visual motion direction appears wrong because the texture's "flame direction" indicator is horizontally flipped on some particles, creating the illusion of reversed motion.

### The Core Tradeoff

We discovered two mutually exclusive states:

1. **No rotation (X=velocity)**: Particles travel correctly (outward), but texture is **upside-down**
2. **With 90° rotation**: Texture orientation is correct, but particles appear **mirrored**

This suggests the problem is fundamental to how GPU axis-based positioning works vs CPU quaternion-based rotation.

## Root Cause Analysis

### GPU vs CPU Billboard Rendering

**CPU VelocityAligned** (quaternion-based):
```rust
let up = velocity_dir;  // Y axis
let forward = Gram-Schmidt(to_camera, up);  // Z axis
let right = up.cross(forward);  // X axis
Quat::from_mat3(&Mat3::from_cols(right, up, forward))
```
The quaternion rotates both vertices AND texture mapping together. When velocity flips, the entire billboard rotates, keeping texture-to-motion relationship consistent.

**GPU AlongVelocity** (axis-based):
```wgsl
let vpos = vertex_position * size;
let sim_position = position + axis_x * vpos.x + axis_y * vpos.y;
// UVs come from mesh, don't change
```
Vertices are positioned by axis vectors, but UVs are fixed to the mesh. When an axis flips (due to cross product sign change), vertex positions flip but UVs stay the same = **horizontal texture mirroring**.

### Why Cross Products Cause Mirroring

For any cross product `cross(A, B)`:
- When B flips sign, the result flips sign
- `cross(A, velocity)` produces opposite results for opposite velocities

Example with `axis_x = cross(axis_y, axis_z)` where axis_y = velocity:
- velocity = (+1, 0, 0), axis_z toward camera: axis_x points some direction
- velocity = (-1, 0, 0), axis_z toward camera: axis_x points **opposite direction**

This causes particles on opposite sides of the explosion to have mirrored texture mapping.

## Attempts Made

### Attempt 1-5: (Previous session - see devlog 042)
Various Gram-Schmidt, cross product order, and world-up reference attempts. All failed.

### Attempt 6: Y=velocity with Camera-Facing Z
**Goal**: Match CPU VelocityAligned exactly (Y=velocity, Z=toward camera, X=Y×Z)
```wgsl
axis_y = normalize(particle.velocity);
let to_camera = normalize(get_camera_position_effect_space() - position);
let z_proj = to_camera - dot(to_camera, axis_y) * axis_y;
axis_z = normalize(z_proj);  // with fallback
axis_x = cross(axis_y, axis_z);
```
**Result**: Still mirrored. The cross product `Y × Z` still flips when velocity flips.

### Attempt 7: Y=velocity with World-Right Projection (No Cross Products)
**Goal**: Avoid cross products entirely for axis_x by projecting a fixed world vector
```wgsl
axis_y = normalize(particle.velocity);

// Project world-right onto plane perpendicular to velocity (Gram-Schmidt)
let world_right = vec3<f32>(1.0, 0.0, 0.0);
let proj_right = world_right - dot(world_right, axis_y) * axis_y;
let proj_right_len_sq = dot(proj_right, proj_right);

// Fallback if velocity is along world-X
let world_forward = vec3<f32>(0.0, 0.0, 1.0);
let proj_forward = world_forward - dot(world_forward, axis_y) * axis_y;

axis_x = select(normalize(proj_forward), normalize(proj_right), proj_right_len_sq > 0.001);
axis_z = cross(axis_x, axis_y);
```

**Mathematical Analysis**:
- velocity = (+1, 0, 0): proj_right = 0, use proj_forward = (0,0,1), axis_x = (0,0,1)
- velocity = (-1, 0, 0): proj_right = 0, use proj_forward = (0,0,1), axis_x = (0,0,1)
- Both cases give the **same** axis_x = (0,0,1)

**Expected**: No mirroring since axis_x is consistent

**Actual Result**: Still mirrored!

This is unexpected and suggests the mirroring source is elsewhere.

## Current State

**bevy_hanabi/src/modifier/output.rs** (lines 657-720):
- Y-axis = velocity
- X-axis = world-right projected perpendicular to velocity
- Z-axis = cross(X, Y)
- No camera dependency for axis_x

**particles.rs**:
- Using `main_9x9.png` (original texture)
- No rotation applied
- Only axis_rotation (random spin) enabled

**Observed**: Still mirrored despite mathematically consistent axis_x.

## Key Insights

1. **The problem persists even without cross products for axis_x**: World-right projection should give consistent axis_x, but mirroring still occurs.

2. **The issue may not be in orientation calculation at all**: We've exhausted orientation code changes. The problem might be in:
   - Vertex shader quad construction (`vfx_render.wgsl`)
   - UV coordinate handling
   - Flipbook modifier
   - Texture sampling

3. **GPU and CPU fundamentally differ**: CPU uses quaternion rotation (texture follows mesh), GPU uses axis positioning (texture fixed to mesh). This may be an inherent limitation.

4. **The upside-down texture reveals the tradeoff**:
   - Original texture + no rotation = correct motion, wrong texture orientation
   - Flipped texture or rotation = correct texture, mirrored motion

## Unexplored Areas

### 1. Vertex Shader UV Handling
Check `vfx_render.wgsl` for how UVs are computed and passed to fragment shader.

### 2. Determinant-Based UV Flip
Detect when orientation matrix has negative determinant (mirrored basis) and flip UV.x:
```wgsl
let det = dot(axis_x, cross(axis_y, axis_z));
uv.x = select(1.0 - uv.x, uv.x, det > 0.0);
```

### 3. Mesh Winding Order
Check if Plane3d mesh has specific winding that interacts with orientation.

### 4. Different Mesh
Try a custom mesh with different vertex/UV layout.

### 5. Fragment Shader UV Flip
Apply UV flip in fragment shader based on particle attributes.

## Files Modified

- `/home/gota/ggando/gamedev/bevy_hanabi/src/modifier/output.rs` (lines 657-720)
- `/home/gota/ggando/gamedev/bevy-mass-render/src/particles.rs` (lines 883, 969-970)

## Resources Created

- `assets/textures/premium/ground_explosion/main_9x9_upsidedown.png` - Vertically flipped flipbook
- `resources/042_gram_schmidt_failed/` - Saved code from earlier attempt

## Session 3 Attempts (December 22, 2025)

### Attempt 8: Quaternion-Based Rotation
**Goal**: Use actual quaternion math to rotate from (0,1,0) to velocity direction, matching CPU exactly.
```wgsl
// Compute quaternion from (0,1,0) to velocity using half-angle formula
let d = dot(from_dir, velocity_dir);
let c = cross(from_dir, velocity_dir);
quat = normalize(vec4(c.x, c.y, c.z, 1.0 + d));

// Convert to rotation matrix
// [standard quaternion to matrix conversion]

// Apply camera-facing rotation around velocity axis
```
**Result**: Still mirrored. The quaternion correctly rotates the orientation, but axis_x still ends up pointing in opposite directions for opposite velocities.

### Attempt 9: Camera-Right Projection for axis_x
**Goal**: Derive axis_x from camera-right vector (consistent regardless of velocity) instead of cross product.
```wgsl
axis_y = normalize(particle.velocity);
let camera_right = get_camera_rotation_effect_space()[0];
let x_proj = camera_right - dot(camera_right, axis_y) * axis_y;
axis_x = normalize(x_proj);
axis_z = cross(axis_x, axis_y);
```
**Result**: Still mirrored. While axis_x projection is consistent, axis_z (from cross product) flips for opposite velocities.

### Attempt 10: Camera-Facing axis_z with Flip Correction
**Goal**: After computing axes, flip both axis_x and axis_z if axis_z points away from camera.
```wgsl
let to_camera = normalize(get_camera_position_effect_space() - position);
let flip = dot(axis_z, to_camera) < 0.0;
axis_x = select(axis_x, -axis_x, flip);
axis_z = select(axis_z, -axis_z, flip);
```
**Result**: Still mirrored. After flipping, axis_z is consistent but axis_x is different for different particles.

### Attempt 11: UV Flip Based on Orientation
**Goal**: Detect mirrored orientation and flip UV.x within flipbook frame.
```wgsl
let needs_uv_flip = dot(axis_x, cam_right) < 0.0;
if (needs_uv_flip) {
    let frame_size = 0.125;  // 1/8 for 8x8 flipbook
    let frame_start = floor(out.uv.x / frame_size) * frame_size;
    let local_x = out.uv.x - frame_start;
    out.uv.x = frame_start + frame_size - local_x;
}
```
**Result**: Caused flashing quads and still mirrored. The `#ifdef NEEDS_UV` in generated WGSL didn't work correctly.

### Attempt 12: Fixed Camera-Facing Rotation Math
**Goal**: Fix bug in sin_angle calculation for camera-facing rotation.
```wgsl
// Wrong:
let sin_angle = dot(cross(axis_z, target_z), axis_y);
// Fixed:
let sin_angle = dot(axis_x, target_z);
```
**Result**: Still mirrored. The rotation direction was fixed but the fundamental axis_x difference persists.

### Attempt 13: Revert to Original X=velocity (BREAKTHROUGH!)
**Goal**: Test if original bevy_hanabi AlongVelocity (X=velocity) has mirroring.
```wgsl
let dir = normalize(position - get_camera_position_effect_space());
axis_x = normalize(particle.velocity);
axis_y = cross(dir, axis_x);
axis_z = cross(axis_x, axis_y);
```
**Result**: NO MIRRORING! Texture is sideways (flames horizontal instead of along velocity) but consistent across all particles.

## Root Cause Identified

The original `AlongVelocity` implementation puts **X-axis along velocity**, not Y-axis. This avoids the mirroring because:
1. axis_x = velocity (directly assigned, no cross product)
2. axis_y = cross(camera_dir, velocity) - this flips for opposite velocities, but...
3. axis_z = cross(velocity, axis_y) - the double cross product cancels out the sign flip

The net effect is that axis_x is always consistent (along velocity), and the sign changes in axis_y and axis_z are internally consistent (both flip together = 180° rotation, not reflection).

When we tried to make Y=velocity:
- axis_y = velocity (consistent)
- axis_z = derived from camera (consistent)
- axis_x = cross(axis_y, axis_z) - FLIPS when axis_y flips!

This single sign flip in axis_x causes the texture horizontal mirroring.

## Solution: Use Original + Rotation

The fix is to use the original X=velocity orientation (which doesn't mirror) and apply a **-90° rotation** in the X-Y plane to get Y along velocity:

```rust
// In particles.rs:
let fireball_rotation = writer_fireball.lit(-std::f32::consts::FRAC_PI_2).expr();

.render(OrientModifier::new(OrientMode::AlongVelocity)
    .with_rotation(fireball_rotation)  // -90° to put Y along velocity
    .with_axis_rotation(fireball_axis_rotation))
```

The rotation formula in bevy_hanabi:
```wgsl
axis_x = axis_x0 * cos(rot) + axis_y0 * sin(rot);
axis_y = axis_y0 * cos(rot) - axis_x0 * sin(rot);
```

For rot = -π/2:
- axis_x = axis_y0 (perpendicular to velocity)
- axis_y = axis_x0 = velocity ✓

This gives Y=velocity without the mirroring issue because the rotation is applied uniformly to all particles, preserving the consistent handedness of the original orientation.

## Key Insights from Session 3

1. **The mirroring is caused by axis_x sign flips**: When axis_x is derived from a cross product involving velocity, it flips sign for opposite velocities.

2. **Original bevy_hanabi approach is correct**: X=velocity avoids the issue because axis_x is directly assigned, not derived.

3. **Rotation preserves handedness**: Applying a fixed rotation after the consistent X=velocity orientation gives Y=velocity without introducing sign flips.

4. **All Y=velocity approaches failed**: Whether using quaternion, Gram-Schmidt, camera-right projection, or UV flips - they all still had mirroring because they derived axis_x from a cross product.

5. **The tradeoff was wrong**: It wasn't "upside-down vs mirrored" - it was "X=velocity (correct) vs Y=velocity (broken)".

## Files Modified

- `/home/gota/ggando/gamedev/bevy_hanabi/src/modifier/output.rs` - Reverted to original AlongVelocity
- `/home/gota/ggando/gamedev/bevy-mass-render/src/particles.rs` - No rotation applied

### Additional Attempts (Session 3 continued)

#### Attempt 14: Rotation + UV Flip with axis_x condition
**Goal**: Apply -90° rotation to get Y=velocity, then flip UV.x when axis_x points opposite to camera-right.
**Result**: Mirrored + flashing. The condition `dot(axis_x, cam_right) < 0.0` changes as camera moves, causing unstable flipping.

#### Attempt 15: Rotation + UV Flip with velocity.x condition
**Goal**: Use `velocity.x < 0.0` as flip condition (stable, doesn't depend on camera).
**Result**: Mirrored + flashing + WORSE. For explosion particles, velocities point in all directions. Particles with velocity.x near 0 (moving up/down) flicker rapidly. This approach only works for strictly horizontal velocities.

## Final Conclusion

**The GPU AlongVelocity mirroring issue is fundamentally unsolvable with orientation code changes.**

### The Math Behind the Problem

For Y=velocity orientation, the axes must be:
- axis_y = velocity (directly assigned, consistent)
- axis_z = camera-related direction (consistent)
- axis_x = cross(axis_y, axis_z) - **MUST flip when axis_y flips**

The cross product `cross(axis_y, axis_z)` inherently produces opposite results for opposite velocities because:
- `cross(+v, z) = +x`
- `cross(-v, z) = -x`

This is unavoidable math. When axis_x flips sign for half the particles, the texture appears horizontally mirrored for those particles.

### Why Original X=velocity Works

The original bevy_hanabi AlongVelocity puts X=velocity:
- axis_x = velocity (directly assigned)
- axis_y = cross(dir, velocity) - flips for opposite velocities
- axis_z = cross(velocity, axis_y) - flips for opposite velocities

Key insight: **both axis_y and axis_z flip together**. This is a 180° rotation around axis_x, not a reflection. Texture appears rotated (acceptable for random-oriented flames) but not mirrored.

### Why UV Flip Doesn't Work

UV flip requires a stable condition to determine which particles should be flipped. All attempted conditions fail:

1. `dot(axis_x, cam_right) < 0.0` - Changes as camera moves = flashing
2. `velocity.x < 0.0` - Only works for horizontal velocities; explosion particles have all directions
3. `dot(velocity, world_right) < 0.0` - Same problem as #2
4. Any other per-particle condition - Will have edge cases that cause flashing

### Available Solutions

1. **Use X=velocity (current state)**: No mirroring, but texture is sideways. Rotate the texture asset 90° so flames point along +X in texture space.

2. **Accept mirroring**: For some effects, mirroring isn't visible (symmetric textures, rapid animation, many particles).

3. **Use CPU particles**: CPU uses quaternion rotation which naturally handles orientation correctly.

4. **Different effect design**: Use radial/circular spawn patterns where velocity direction is less critical.

## Current State

Code reverted to original bevy_hanabi AlongVelocity:
- X-axis = velocity direction
- No rotation applied
- Texture appears sideways but consistent (no mirroring, no flashing)

The GPU fireball effect currently shows flames pointing sideways (perpendicular to velocity) but all particles are consistent.
