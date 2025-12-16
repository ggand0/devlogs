# Terrain-Conforming Visual Indicators

**Date:** 2025-12-03
**Branch:** `feat/heightmap-indicator-visuals`

## Problem

Visual indicators (selection rings, movement circles, arrows, group bounding boxes) were always horizontally flat in the XZ plane. On sloped terrain (Map 2: Rolling Hills), they appeared disconnected from the ground, floating or clipping through terrain.

## Solution

Implemented heightmap-based terrain conformance for all visual indicators by sampling terrain height at multiple points:

### Selection Rings & Movement Circles
- **32-segment subdivision** around the ring
- Each vertex samples terrain height at its world XZ position
- Mesh stored in local space, Y-relative to center height
- Regenerates when squad moves >0.1 units

### Movement Arrows
- Sample terrain height at arrow base position
- Arrows now properly positioned on slopes

### Group Bounding Box (Debug Visualization)
- **16-segment subdivision** along each edge (front, back, left, right)
- Each segment samples terrain at its XZ position
- Creates cylindrical line meshes (6 segments around radius)
- Despawn/recreate pattern for updates

### Group Orientation Marker
- Yellow triangle arrow samples terrain at its base position
- Properly positioned on slopes

## Technical Implementation

**Key Function:**
```rust
fn create_line_mesh(
    start: Vec3,
    end: Vec3,
    thickness: f32,
    heightmap: Option<&TerrainHeightmap>
) -> Mesh
```

**Critical Fix:** Actually sample heightmap at each segment point instead of linear interpolation:
```rust
let center_y = if let Some(hm) = heightmap {
    hm.sample_height(center_x, center_z) + 0.2
} else {
    start.y + (end.y - start.y) * t
};
```

## Files Changed

- `src/selection/visuals/selection.rs` - Selection ring terrain sampling
- `src/selection/visuals/movement.rs` - Movement circle and arrow terrain sampling
- `src/selection/visuals/group.rs` - Group bounding box and orientation marker
- `src/terrain.rs` - Added `sample_normal()` and `sample_height_and_normal()` (infrastructure, unused in current implementation)
- `src/decals.rs` - Extended DecalTextures resource with selection_ring field (infrastructure for future texture-based approach)

## Performance

No noticeable performance impact. Heightmap sampling is O(1) bilinear interpolation. The procedural mesh approach works well at current scale.

## Future Improvements

The original plan included using **textured quad decals** (with `WFX_T_GlowCircle A8.png`) aligned to terrain normals for even better visual quality and batching. The current procedural mesh approach works well, but texture-based decals would be the next optimization if needed.

## Testing

- ✅ Flat terrain (Map 1) - No visual regressions
- ✅ Rolling hills (Map 2) - Indicators conform to slopes
- ✅ Multiple squad selections - Works correctly
- ✅ Group bounding box - Edges follow terrain contours
- ✅ Squad movement - Indicators update smoothly

## Result

All visual indicators now properly follow terrain contours. No more floating or clipping indicators on sloped terrain!
