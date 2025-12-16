# Laser Billboarding Singularity Fix

**Date:** June 6, 2025  
**Issue:** Laser orientation artifacts when camera looks straight down  
**Status:** ✅ Fixed

---

## Problem Description

### Initial Bug Report
User noticed a critical visual bug with laser projectiles in the RTS demo:
- When tilting the camera to look straight down at the battlefield, lasers displayed weird orientation artifacts
- The long axis of lasers started having strange angles as they got closer to what appeared to be the "center" of the camera view
- Described as an "invisible circle" effect where lasers flying near this center point had tangential orientations
- The issue was **not** present when looking at the battlefield from normal angles
- Lasers at the edge of the camera view appeared fine, only those near the center were affected

### Technical Symptoms
- Circular pattern of mis-oriented lasers around a focal point when camera is nearly vertical
- Severity correlated with the angle between the Y-axis up vector and the camera direction
- Billboard facing (quad visibility) was correct
- **Only the long axis alignment** with velocity direction was broken
- The closer the camera looked straight down (to_camera.dot(Vec3::Y) approaching 1.0), the worse the artifacts

---

## Root Cause Analysis

The bug was caused by a **mathematical singularity** in the laser orientation calculation when using a fixed world-space reference vector (`Vec3::Y`) for aligning the laser's long axis.

### The Original Problematic Code

```rust
fn calculate_laser_orientation(
    velocity: Vec3,
    position: Vec3,
    camera_position: Vec3,
) -> Quat {
    if velocity.length() > 0.0 {
        let velocity_dir = velocity.normalize();
        let to_camera = (camera_position - position).normalize();
        
        // First, make the quad face the camera
        let base_rotation = Transform::from_translation(Vec3::ZERO)
            .looking_at(-to_camera, Vec3::Y)  // ❌ PROBLEM: Fixed Vec3::Y
            .rotation;
        
        // Then rotate around the camera direction to align with velocity
        let velocity_in_quad_plane = velocity_dir - velocity_dir.dot(to_camera) * to_camera;
        if velocity_in_quad_plane.length() > 0.001 {
            let velocity_in_quad_plane = velocity_in_quad_plane.normalize();
            let angle = Vec3::Y.dot(velocity_in_quad_plane).acos();  // ❌ PROBLEM: Fixed Vec3::Y reference
            let cross = Vec3::Y.cross(velocity_in_quad_plane);
            let rotation_sign = if cross.dot(to_camera) > 0.0 { 1.0 } else { -1.0 };
            
            let alignment_rotation = Quat::from_axis_angle(to_camera, angle * rotation_sign);
            alignment_rotation * base_rotation
        } else {
            base_rotation
        }
    } else {
        Quat::IDENTITY
    }
}
```

### Why It Failed

1. **Billboard Creation Issue**: When `to_camera` is nearly parallel to `Vec3::Y` (camera looking straight down), the `looking_at(-to_camera, Vec3::Y)` call creates a degenerate case where the up vector and view direction are nearly parallel

2. **Fixed Reference Problem**: Using `Vec3::Y` as a fixed reference for calculating the rotation angle created a circular dependency:
   - Each laser's orientation depended on its position relative to the camera
   - When the camera looked down, positions radially distributed around the camera focus created a circular pattern of mis-orientations
   - The "up" direction meant different things for different lasers depending on their position

3. **Inconsistent Coordinate System**: The billboard's actual "up" direction after rotation didn't match the fixed `Vec3::Y` reference used for velocity alignment

---

## Attempted Solutions (Failed)

### Attempt 1: Manual Billboard Matrix Construction
**Approach:** Manually construct the billboard rotation matrix using cross products instead of `looking_at()`
- Detected when camera was nearly vertical and switched up vector from Y to X
- Manually built rotation matrix to avoid `looking_at()` singularity

**Result:** ❌ Failed - Issue persisted

### Attempt 2: Simplified Camera-Independent Orientation
**Approach:** Removed per-frame rotation updates and used world-space-only velocity alignment
- Aligned laser long axis with velocity using consistent world-space references
- Completely ignored camera position in orientation calculation

**Result:** ❌ Failed - Quads became too thin from certain viewing angles (broke billboarding)

### Attempt 3: Quat::from_rotation_arc Approach
**Approach:** Used `Quat::from_rotation_arc()` instead of `Transform::looking_at()` with dynamic up vector selection

**Result:** ❌ Failed - Issue persisted

### Attempt 4: Complete Billboarding Abandonment
**Approach:** Gave up on camera-dependent billboarding entirely, used only velocity alignment

**Result:** ❌ Failed - Lasers appeared as thin lines from many angles

### Attempt 5: Hybrid World-Space Billboard
**Approach:** Created billboard coordinate system using world-space projections

**Result:** ❌ Failed - Broke billboarding AND didn't fix the singularity

---

## The Solution

### Key Insight
The problem wasn't the billboard creation itself, but using **`Vec3::Y` as the reference for velocity alignment** instead of **the billboard's actual "up" direction**.

### Fixed Code

```rust
fn calculate_laser_orientation(
    velocity: Vec3,
    position: Vec3,
    camera_position: Vec3,
) -> Quat {
    if velocity.length() > 0.0 {
        let velocity_dir = velocity.normalize();
        let to_camera = (camera_position - position).normalize();
        
        // Choose a stable up vector for billboarding that's not parallel to to_camera
        let up = if to_camera.dot(Vec3::Y).abs() > 0.95 {
            Vec3::X // fallback when camera is nearly vertical
        } else {
            Vec3::Y // normal case
        };
        
        // First, make the quad face the camera using stable up vector
        let base_rotation = Transform::from_translation(Vec3::ZERO)
            .looking_at(-to_camera, up)
            .rotation;
        
        // ✅ KEY FIX: Calculate the billboard's actual "up" direction after rotation
        let billboard_up = base_rotation * Vec3::Y;
        
        // Project velocity onto the billboard plane
        let velocity_in_quad_plane = velocity_dir - velocity_dir.dot(to_camera) * to_camera;
        if velocity_in_quad_plane.length() > 0.001 {
            let velocity_in_quad_plane = velocity_in_quad_plane.normalize();
            
            // ✅ Use billboard's actual up direction instead of fixed Vec3::Y
            let angle = billboard_up.dot(velocity_in_quad_plane).acos();
            let cross = billboard_up.cross(velocity_in_quad_plane);
            let rotation_sign = if cross.dot(to_camera) > 0.0 { 1.0 } else { -1.0 };
            
            let alignment_rotation = Quat::from_axis_angle(to_camera, angle * rotation_sign);
            alignment_rotation * base_rotation
        } else {
            base_rotation
        }
    } else {
        Quat::IDENTITY
    }
}
```

### What Changed

1. **Dynamic Up Vector Selection** (threshold at 0.95):
   - Switches from `Vec3::Y` to `Vec3::X` when camera is nearly vertical
   - Prevents `looking_at()` singularity

2. **Billboard-Relative Reference** (the critical fix):
   - Computes `billboard_up = base_rotation * Vec3::Y`
   - Uses this billboard-relative "up" direction for velocity alignment
   - Eliminates the circular dependency on world-space Y-axis

3. **Updated in `update_projectiles` System**:
   - Changed to call `calculate_laser_orientation()` every frame
   - Ensures consistent orientation updates throughout projectile lifetime

---

## Technical Explanation

### The Math Behind the Fix

**Before (Broken):**
```
angle = Vec3::Y.dot(velocity_in_quad_plane).acos()
```
- Used fixed world Y-axis as reference
- Different lasers at different positions had different relationships to world Y
- Created position-dependent artifacts in a circular pattern

**After (Fixed):**
```
let billboard_up = base_rotation * Vec3::Y;
angle = billboard_up.dot(velocity_in_quad_plane).acos()
```
- Uses billboard's local Y-axis (which varies per laser based on camera angle)
- Consistent reference frame for all lasers regardless of position
- Eliminates circular artifacts

### Why This Works

The billboard's "up" direction is computed **in the billboard's local coordinate system**, which already accounts for:
- The laser's position relative to camera
- The camera's orientation
- The chosen up vector (Y or X fallback)

By using this local reference instead of a global one, each laser's velocity alignment is computed in a **consistent local space** rather than trying to reconcile global and local coordinate systems.

---

## Testing & Verification

### Test Procedure
1. Run demo: `VK_LOADER_DEBUG=error VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/radeon_icd.x86_64.json cargo run --release`
2. Tilt camera to look straight down at battlefield (maximum pitch)
3. Press 'F' to trigger volley fire
4. Observe laser orientations, especially near center of screen

### Results
- ✅ No circular artifacts when looking straight down
- ✅ Long axis correctly aligns with velocity direction at all camera angles
- ✅ Proper billboarding maintained (quads face camera)
- ✅ Smooth transitions when moving camera from oblique to vertical angles
- ✅ No performance impact

---

## Lessons Learned

1. **Singularities in 3D Math**: Always check for degenerate cases when using `looking_at()`, `cross()`, or `normalize()` with vectors that might become parallel

2. **Coordinate System Consistency**: When mixing world-space and view-space calculations, ensure reference vectors belong to the same coordinate system

3. **Local vs Global References**: Billboard-relative calculations should use billboard-relative reference vectors, not world-space ones

4. **Debugging Spatial Issues**: The "invisible circle" pattern was a strong hint that the issue was related to radial distance/angle from a focal point (the camera)

5. **Simple Solutions**: After 5 failed attempts with complex approaches, the fix was a single line: `let billboard_up = base_rotation * Vec3::Y`

---

## Related Files Modified

- `src/main.rs`: 
  - `calculate_laser_orientation()` function
  - `update_projectiles()` system

---

## Performance Notes

- No measurable performance impact
- Calculation complexity remains O(1) per projectile
- Still maintains 40+ FPS with 10,000 units and hundreds of active projectiles

---

## Future Considerations

- Could potentially optimize by caching billboard_up calculations if multiple lasers share same camera orientation
- Consider applying similar fix to any other billboard-based effects (explosions, particle systems, etc.)
- Document this pattern for future billboard implementations

---

**Status:** Issue resolved and tested ✅  
**Commit:** "Fix laser billboarding singularity with dynamic up vector reference"

