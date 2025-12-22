# GPU Fireball Debug Analysis - Session 4

**Date**: December 22, 2025
**Status**: Resolved - Root cause confirmed
**Branch**: feat/gpu-particle-perf

## Session Goal

Add debug visualization to understand WHY the quaternion and rotation approaches fail for GPU particle orientation.

## Debug Approach

Added color-based axis visualization to the GPU shader:
```wgsl
color = vec4<f32>(
    axis_x.x * 0.5 + 0.5,  // R = axis_x.x mapped [-1,1] to [0,1]
    axis_x.y * 0.5 + 0.5,  // G = axis_x.y
    axis_x.z * 0.5 + 0.5,  // B = axis_x.z
    1.0
);
```

**Result**: Particles displayed **rainbow of colors** - confirming axis_x is different for each velocity direction.

## Mathematical Analysis

### Why Quaternion Rotation Fails

For quaternion from (0,1,0) to velocity using half-angle formula:
```
q = normalize(cross(from, to), 1 + dot(from, to))
```

**velocity = (+1, 0, 0)**:
- cross((0,1,0), (1,0,0)) = (0, 0, -1)
- Quaternion rotates -90° around Z axis
- After rotation: axis_x = (0, **-1**, 0)

**velocity = (-1, 0, 0)**:
- cross((0,1,0), (-1,0,0)) = (0, 0, +1)
- Quaternion rotates +90° around Z axis
- After rotation: axis_x = (0, **+1**, 0)

**Conclusion**: Shortest-path quaternion rotation takes different paths for opposite velocities, producing opposite axis_x directions.

### Why -90° Rotation Approach Fails

The rotation formula:
```wgsl
axis_x = axis_x0 * cos(rot) + axis_y0 * sin(rot);
axis_y = axis_y0 * cos(rot) - axis_x0 * sin(rot);
```

For rot = -π/2:
- axis_y = axis_x0 = velocity (consistent ✓)
- axis_x = -axis_y0 = -cross(dir, velocity) (flips when velocity flips ✗)

The rotation swaps X and Y axes, but axis_x inherits the sign flip from axis_y0. The mirroring is transferred, not eliminated.

## Root Cause Confirmed

For any Y=velocity orientation, axis_x must be derived from something. Whether:
1. Cross product with velocity → flips
2. Quaternion rotation from reference direction → flips
3. Rotated axis that was originally cross-product derived → flips
4. Gram-Schmidt projection → involves cross product → flips

**There is no way to derive axis_x without it flipping for opposite velocities.**

The ONLY exception is when axis_x = velocity directly (X=velocity orientation), because velocity is assigned, not derived.

## Why X=velocity Works

Original bevy_hanabi AlongVelocity:
```wgsl
axis_x = normalize(particle.velocity);  // Directly assigned
axis_y = cross(dir, axis_x);            // Flips for opposite velocities
axis_z = cross(axis_x, axis_y);         // Flips for opposite velocities
```

When velocity flips:
- axis_x: consistent (directly assigned)
- axis_y: flips
- axis_z: flips

Two axes flipping together = 180° rotation around axis_x = even parity = no mirroring.
The texture appears rotated but not mirrored.

## Why Y=velocity Always Mirrors

Any Y=velocity approach:
```wgsl
axis_y = normalize(particle.velocity);  // Directly assigned
axis_z = some_camera_direction;         // Consistent
axis_x = cross(axis_y, axis_z);         // FLIPS!
```

When velocity flips:
- axis_y: flips (velocity)
- axis_z: consistent
- axis_x: flips (cross product result)

Two axes flipping, one consistent = odd parity relative to X=velocity case.
This causes the texture horizontal mirroring.

## Solution

**For GPU particles with velocity-aligned orientation without mirroring:**

1. Use X=velocity (original bevy_hanabi AlongVelocity) - no mirroring
2. Rotate the texture asset 90° so flames point along +X instead of +Y

This is the only solution that works for arbitrary velocity directions.

## Files Modified

- `/home/gota/ggando/gamedev/bevy_hanabi/src/modifier/output.rs` - Reverted to original X=velocity
- `/home/gota/ggando/gamedev/bevy-mass-render/src/particles.rs` - Removed rotation, X=velocity only

## Current State

- GPU fireball uses X=velocity orientation
- No mirroring, no flashing
- Texture appears sideways (flames perpendicular to velocity)
- Next step: rotate texture asset 90° for correct visual appearance

## Key Learnings

1. **Debug visualization is essential** - Color-coding axis values immediately revealed the problem
2. **Quaternion shortest-path is the culprit** - Different paths to opposite targets produce different intermediate axes
3. **Cross products propagate sign** - Any axis derived from a cross product with velocity will flip
4. **Even vs odd parity** - Two axes flipping = rotation (OK), one axis flipping = reflection (mirroring)
5. **The solution is in the texture, not the code** - Rotate the asset to match X=velocity orientation

## Implementation Plan: Texture Rotation

### Why This Is Different From Previous Texture Flip

**Previous attempt (failed)**:
- Used Y=velocity orientation with -90° rotation in code
- Code had mirroring bug (axis_x inconsistent)
- Flipping texture vertically didn't help because the CODE was broken

**Current approach (will work)**:
- Use X=velocity orientation (original bevy_hanabi AlongVelocity)
- Code has NO mirroring (axis_x = velocity, always consistent)
- Texture appears sideways because flames point +Y in texture, but velocity is +X
- Rotate texture 90° clockwise → flames point +X → aligns with velocity direction

The texture rotation isn't fixing a code bug - it's adapting to a different coordinate convention.

### Steps

1. Rotate `main_9x9.png` 90° clockwise to create `main_9x9_xvel.png`
2. Update `particles.rs` to use the rotated texture for GPU fireball
3. Test to confirm no mirroring and correct flame direction

### Expected Result

- GPU fireball particles travel outward from explosion center
- Flames point along velocity direction (not sideways)
- No mirroring, no flashing
- Consistent appearance across all velocity directions
