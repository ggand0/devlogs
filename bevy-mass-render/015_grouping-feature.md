# Add Total War-style squad grouping with formation preservation

Branch: `refactor/explosion-modules-cleanup`

## Summary
Implements Total War-style squad grouping where multiple selected squads maintain their relative formation positions across move commands. Groups persist until manually ungrouped, enabling tactical multi-squad maneuvers.

## Commit History

| Commit | Description |
|--------|-------------|
| `af674ba` | Add Total War-style squad grouping (G to toggle, U to ungroup) |
| `3125397` | Add yellow selection ring for grouped squads |
| `443a6de` | Add visual orientation indicator for squad groups |
| `2c85254` | WIP: Fix group rotation direction (attempt 1) |
| `2429b31` | Fix group rotation to preserve formation shape across multiple moves |
| `15b6510` | WIP: Add debug bounding box visualization for group orientation |
| `e454e12` | Fix OBB visualization to rotate with group facing direction |
| `dc8ed39` | Add nested debug shortcuts with UI indicator |
| `4b3d03c` | Extract explosion_system.rs from objective.rs |
| `5a6aa8f` | Clean up explosion_shader.rs: remove dead code |
| `4b02d69` | Fix unsafe static mut in formation.rs |
| `TBD` | Add persistent path arrows for selected moving squads |
| `TBD` | Fix path arrows not disappearing when squads arrive |
| `TBD` | Split visuals.rs into focused submodules (selection/movement/group) |
| `TBD` | Consolidate duplicate arrow mesh functions |
| `TBD` | Extract utility functions and constants for arrival detection |

## Features

### Squad Grouping (G/U keys)
- **G key**: Toggle grouping for 2+ selected squads
- **U key**: Ungroup selected squads
- Groups preserve relative squad positions in original coordinate system
- Grouped squads show **yellow selection rings** (vs cyan for ungrouped)
- Selecting any squad in a group auto-selects the entire group

### Formation Preservation
- Groups store `original_formation_facing` (immutable) and squad offsets from group center
- Move commands rotate offsets from original facing to new facing direction
- Prevents formation distortion across multiple move orders (fixed compound rotation bug)
- OBB (Oriented Bounding Box) visualization rotates with group facing

### Visual Feedback
- **Yellow triangle arrow**: Group orientation indicator at front edge of bounding box
- **Magenta OBB wireframe**: Debug visualization showing rotated bounding box (comment out line in main.rs to disable)
- **Green path arrows**: Persistent arrows showing movement direction for selected squads
  - Automatically appear when selected squads are moving toward a destination
  - Disappear when squads arrive (distance < 5.0 units)
  - Update in real-time as squads move
- **Move indicators**: Green circles and path lines that fade out after move commands
- Smooth interpolation for orientation updates (reduces visual jitter)

## Technical Details

### Core Components
```rust
pub struct SquadGroup {
    pub id: u32,
    pub squad_ids: Vec<u32>,
    pub squad_offsets: HashMap<u32, Vec3>,       // Relative offsets (original coords)
    pub original_formation_facing: Vec3,          // Immutable reference direction
    pub formation_facing: Vec3,                   // Current facing (for visuals)
}

pub struct OrientedBoundingBox {
    pub center: Vec3,
    pub half_extents: Vec2,     // Half-width and half-depth
    pub facing: Vec3,           // Forward direction
    pub right: Vec3,            // Perpendicular direction
}
```

### Rotation Algorithm
1. Store offsets relative to group center when grouping
2. Store `original_formation_facing` (never changes)
3. On move: calculate rotation angle from original to new facing
4. Apply rotation to stored offsets: `rotated_offset = rotation * original_offset`
5. Position squads at: `destination + rotated_offset`

This ensures formation shape is preserved by always rotating from the same base coordinate system.

### Module Structure
Refactored monolithic 1850-line `selection.rs` into focused submodules:
- `state.rs`: SelectionState resource and marker components
- `groups.rs`: Squad grouping logic
- `input.rs`: Click and box selection handling
- `movement.rs`: Right-click move commands with orientation drag
- `visuals/`: Visual feedback systems (split from 969-line visuals.rs)
  - `selection.rs`: Selection rings and box selection visuals
  - `movement.rs`: Move indicators, path lines, orientation arrows, persistent path arrows
  - `group.rs`: Group orientation markers and OBB debug visualization
  - `mod.rs`: Re-exports for clean public API
- `obb.rs`: Oriented Bounding Box calculations
- `utils.rs`: Shared utility functions (`horizontal_distance`, `horizontal_direction`, etc.)

### OrientedBoundingBox (OBB)
Replaced axis-aligned bounding box (AABB) with OBB for proper rotation:
- `from_squads()`: Calculates OBB aligned to facing direction
- `corners()`: Returns 4 world-space corners
- `front_edge_center()`: Position for orientation indicator

## How It Works

1. **Group Creation**: When 2+ squads are selected and G is pressed:
   - Store each squad's offset from group center
   - Store the average facing direction as `original_formation_facing`
   - Map squads to group ID in `squad_to_group`

2. **Group Movement**: On move command:
   - Calculate rotation from `original_formation_facing` to new facing
   - Apply rotation to each squad's stored offset
   - Move squads to `destination + rotated_offset`

3. **Visualization**:
   - Yellow selection rings on grouped squads (instead of cyan)
   - Yellow triangle arrow at front edge of OBB
   - Magenta wireframe rectangle showing OBB bounds (debug)

## Key Fixes

### Formation Distortion (commit `2429b31`)
**Problem**: Group shape changed after multiple move orders
**Cause**: Rotation calculated from current facing, causing compound errors
**Fix**: Store immutable `original_formation_facing`, always rotate from this base

### Inverted Rotation (`2c85254`)
**Problem**: Groups rotated opposite to intended direction
**Fix**: Correct `atan2(x, z)` for XZ plane angles from +Z axis

### Bounding Box Bugs
- **Thin line bug** (`15b6510`): Used unit cube (1×1×1) + scale instead of baked dimensions
- **AABB not rotating** (`e454e12`): Implemented OBB with local-space coordinate transformation

### Unsafe Static Mutation (`4b02d69`)
**Problem**: Used `static mut LAST_UPDATE_TIME: f32` with unsafe block in [formation.rs:66-76](../src/formation.rs#L66-L76)
**Fix**: Replace with Bevy's `Local<f32>` system parameter for safe per-system state

### Path Arrows Not Disappearing
**Problem**: Persistent path arrows kept showing even after squads arrived at destination
**Cause**: Inconsistent arrival thresholds - path arrow used 2.0, formation system used 5.0
**Fix**:
- Extract `SQUAD_ARRIVAL_THRESHOLD = 5.0` constant to [constants.rs:61](../src/constants.rs#L61)
- Use consistent threshold across formation.rs and visuals/movement.rs

## Code Quality Improvements

### Refactoring: visuals.rs → visuals/ module (969 → 3 files)
Split monolithic visuals.rs into focused submodules for better maintainability:
- **selection.rs** (202 lines): Selection ring and box selection visuals
- **movement.rs** (407 lines): Move indicators, path lines, arrows
- **group.rs** (311 lines): Group orientation markers and OBB debug viz
- **mod.rs** (21 lines): Clean re-exports

### Consolidated Arrow Mesh Functions
Replaced two nearly-identical `create_orientation_arrow_mesh()` and `create_path_arrow_mesh()` with single parameterized function:
```rust
pub fn create_arrow_mesh(shaft_width: f32, head_width: f32, head_length: f32) -> Mesh
```

### Utility Functions
Added reusable utilities to [utils.rs](../src/selection/utils.rs):
```rust
#[inline]
pub fn horizontal_distance(a: Vec3, b: Vec3) -> f32  // ~10 call sites
pub fn horizontal_direction(from: Vec3, to: Vec3) -> Vec3  // ~8 call sites
```
Replaced manual `Vec3::new(a.x - b.x, 0.0, a.z - b.z).length()` patterns across the module.

## Controls Summary
- **Left-click**: Select squad (or entire group if grouped)
- **Shift+click**: Add/remove from selection
- **Box select**: Drag left-click to select multiple squads
- **Right-click drag**: Move + set orientation (CoH1-style)
- **G**: Group selected squads (toggle)
- **U**: Ungroup selected squads
- **T**: Advance all units (changed from G to avoid conflict)

## Debug Features
- Magenta OBB wireframe appears when complete group is selected
- `info!` logging shows OBB dimensions on group creation
- `Ctrl+Shift+1-5`: Test explosion effects (unrelated feature)
