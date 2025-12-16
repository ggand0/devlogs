# Turret Model Redesign

## Overview
Replaced the initial placeholder turret model with a more detailed sci-fi design. The original model was a simple collection of cylinders and boxes that didn't look like a functional turret.

## Changes

### 1. Base Structure (`create_turret_base_mesh`)
- **Old:** Simple square box.
- **New:** Reinforced concrete bunker style.
  - **Foundation:** Octagonal slab.
  - **Body:** Sloped frustum (truncated pyramid shape) for projectile deflection.
  - **Ring:** Detailed turret ring at the top.
- **Implementation:** Added `add_frustum` helper to generate tapered cylinders/cones with correct normals.

### 2. Rotating Assembly (`create_turret_rotating_assembly_mesh`)
- **Old:** Upright cylinder with vertical barrels (looked like a chimney).
- **New:** Articulated gun turret.
  - **Housing:** Horizontal armored box, slightly rear-weighted.
  - **Barrels:** Dual heavy cannons with muzzle brakes, extending horizontally.
  - **Armor:** Side "cheek" armor pods.
  - **Sensors:** Optical/Radar pod on top.
- **Orientation:**
  - Barrels now point towards `-Z` (negative Z).
  - This aligns with Bevy's default `Transform::look_at` behavior (which points -Z at the target).
  - Previous implementation likely pointed +Z or +Y, causing the turret to look away from the target or point upwards.

### 3. Geometry Helpers
- Added `add_frustum` for tapered shapes.
- Added `add_z_cylinder` for horizontal barrels.
- Consolidated geometry logic within the mesh generation functions.

## Visual Goal
The new turret looks like a dedicated point-defense system, grounded firmly in the terrain, with a functional rotating head that tracks targets correctly.
