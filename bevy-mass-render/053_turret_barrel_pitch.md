# 053: MG Turret Barrel Pitch Rotation

## Overview

Added independent barrel pitch rotation for MG turrets. The barrel now aims up/down at targets while the assembly only rotates horizontally (yaw).

## Problem

Previously, the entire turret assembly would tilt when attempting to add pitch rotation, which looked unnatural. Turrets should have:
- Assembly: horizontal rotation (yaw) to face target
- Barrel: vertical rotation (pitch) to aim at target elevation

## Solution

Split the barrel into a separate child entity with its own transform, creating a 3-level hierarchy:

```
TurretBase (stationary)
  └── TurretRotatingAssembly (yaw rotation)
        └── TurretBarrel (pitch rotation)
```

## Implementation

### 1. Barrel Mesh Separation (procedural_meshes.rs)

**Removed from `create_mg_turret_assembly_mesh()`:**
- 4 Z-cylinder stages that formed the barrel

**New function `create_mg_turret_barrel_mesh()`:**
```rust
// Barrel geometry centered at pivot point (0, 0, 0)
// Stage 1: Base Connector/Shroud at Z=-0.2
// Stage 2: Recoil Spring at Z=-1.2
// Stage 3: Main Barrel at Z=-4.0
// Stage 4: Muzzle Brake at Z=-6.2
```

Barrel mesh is centered at its pivot point so rotation works correctly.

### 2. TurretBarrel Component (types.rs)

```rust
/// Marker component for MG turret barrel (child of TurretRotatingAssembly)
/// The barrel entity handles pitch rotation while the assembly handles yaw
#[derive(Component)]
pub struct TurretBarrel;
```

### 3. Entity Hierarchy (turrets.rs)

```rust
// Spawn barrel entity (child of assembly) - handles PITCH rotation
let barrel_entity = commands.spawn((
    Mesh3d(barrel_mesh),
    MeshMaterial3d(barrel_material),
    Transform::from_xyz(0.0, 2.0, -1.0), // Pivot at weapon housing front
    TurretBarrel,
)).id();

// Link: Base -> Assembly -> Barrel
commands.entity(base_entity).add_children(&[assembly_entity]);
commands.entity(assembly_entity).add_children(&[barrel_entity]);
```

Barrel positioned at `(0, 2, -1)` relative to assembly - the front edge of the weapon housing.

### 4. Rotation System (combat.rs)

**turret_rotation_system** now handles both:

```rust
// YAW: Assembly rotation (horizontal)
let target_pos_flat = Vec3::new(target_pos.x, turret_pos.y, target_pos.z);
let target_rotation = Transform::from_translation(turret_pos)
    .looking_at(target_pos_flat, Vec3::Y)
    .rotation;
turret_transform.rotation = turret_transform.rotation.slerp(target_rotation, ...);

// PITCH: Barrel rotation (vertical) - MG only
if mg_turret.is_some() {
    for child in children.iter() {
        if let Ok(mut barrel_transform) = barrel_query.get_mut(child) {
            let pitch_angle = vertical_dist.atan2(horizontal_dist);
            let pitch_clamped = pitch_angle.clamp(-0.52, 0.26); // -30° to +15°
            let target_pitch = Quat::from_rotation_x(pitch_clamped);
            barrel_transform.rotation = barrel_transform.rotation.slerp(target_pitch, ...);
        }
    }
}
```

### 5. Firing Position (combat.rs)

**turret_hitscan_fire_system** computes muzzle position with pitch:

```rust
let firing_pos = if is_mg {
    // Barrel pivot position
    let barrel_pivot_local = Vec3::new(0.0, 2.0, -1.0);
    let barrel_pivot_world = assembly_pos + local_transform.rotation * barrel_pivot_local;

    // Calculate pitch to target
    let to_target = target_pos - barrel_pivot_world;
    let pitch_angle = vertical_dist.atan2(horizontal_dist).clamp(-0.52, 0.26);

    // Apply pitch to get muzzle position
    let barrel_forward = local_transform.rotation * Vec3::NEG_Z;
    let pitch_rotation = Quat::from_axis_angle(local_transform.rotation * Vec3::X, pitch_angle);
    let pitched_forward = pitch_rotation * barrel_forward;
    barrel_pivot_world + pitched_forward * 6.4  // Muzzle 6.4 units from pivot
} else {
    // Heavy turret: standard barrel positions
    ...
};
```

## Pitch Limits

| Direction | Angle | Radians |
|-----------|-------|---------|
| Down (max) | -30° | -0.52 |
| Up (max) | +15° | +0.26 |

## Heavy Turret

Heavy turret still uses dual fixed barrels without pitch. The barrel positions remain hardcoded:
- Left: `(-1.8, 1.5, -6.0)`
- Right: `(1.8, 1.5, -6.0)`

Future work could add pitch to heavy turret by splitting its barrels similarly.

## Files Modified

| File | Changes |
|------|---------|
| `src/procedural_meshes.rs` | Split barrel from assembly, new `create_mg_turret_barrel_mesh()` |
| `src/types.rs` | Added `TurretBarrel` component |
| `src/turrets.rs` | 3-level entity hierarchy, barrel material |
| `src/combat.rs` | Pitch in rotation system, dynamic muzzle position calculation |

## Result

- MG turret barrel pitches up/down independently
- Assembly only rotates horizontally
- Tracers fire from correct muzzle position regardless of pitch
- Smooth interpolation for natural movement
