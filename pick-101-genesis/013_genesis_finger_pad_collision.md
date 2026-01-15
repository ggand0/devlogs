# Genesis Finger Pad Collision Investigation

## Problem Statement

In MuJoCo, the SO-101 gripper required finger pad collision boxes (2.5mm cubes at fingertips) to achieve stable grasping. Without them, mesh collision at fingertips causes single unstable contact points, leading to cube slipping during lift.

Reference: [MuJoCo GitHub Issue #239](https://github.com/google-deepmind/mujoco/issues/239), devlog 021

The finger pad boxes were added to `so101_new_calib.xml` and work in MuJoCo. This investigation tests whether they work in Genesis.

## Test Setup

### Test Script: `tests/test_grasp_stability.py`

Uses `LiftCubeEnv` with IK controller matching MuJoCo implementation:

1. **Phase 0-1**: Initialize arm, let cube settle
2. **Phase 2**: Move TCP above cube with IK (`locked_joints=[3,4]` for top-down orientation)
3. **Phase 3**: Lower to grasp height
4. **Phase 4**: Close gripper gradually
5. **Phase 5**: Lift with IK
6. **Phase 6**: Hold and measure stability

### Grasp Parameters
```python
height_offset = 0.03      # Height above cube for approach
grasp_z_offset = -0.01    # TCP target relative to cube center
finger_width_offset = -0.015  # X-axis offset for finger centering
GRIPPER_OPEN = 0.5
GRIPPER_CLOSED = -0.15
```

## Findings

### 1. Finger Pad Boxes ARE Loaded by Genesis

```
Robot geoms count: 32
  Geom 28: type=<BOX: 5>, link=gripper, friction=1.0, contype=1
  Geom 31: type=<BOX: 5>, link=moving_jaw_so101_v1, friction=1.0, contype=14
```

The box geoms exist with correct friction (1.0) and positions matching MJCF.

### 2. Box Pads Are NOT Making Contact

Verified by querying `env.cube.get_contacts()` which returns `geom_a` and `geom_b` indices for each contact pair.

Geom index mapping (verified at runtime):
```
geom.idx=30, type=<MESH>, link=gripper
geom.idx=31, type=<MESH>, link=gripper
geom.idx=32, type=<MESH>, link=gripper
geom.idx=33, type=<MESH>, link=gripper
geom.idx=34, type=<BOX>, link=gripper          # Static finger pad
geom.idx=35, type=<MESH>, link=moving_jaw_so101_v1
geom.idx=36, type=<MESH>, link=moving_jaw_so101_v1
geom.idx=37, type=<BOX>, link=moving_jaw_so101_v1  # Moving finger pad
geom.idx=38 = cube
```

Contact query during gripper close shows:
```
      ext:38 <-> gripper:MESH   (indices 30-33)
      ext:38 <-> gripper:MESH
  Step 120: gripper=0.119, cube_z=0.0215, contacts=18
```

All 18 contacts are between cube (idx=38) and mesh geoms (idx=30-33, 35-36).
**Zero contacts with box geoms (idx=34, 37)**.

### 3. Box Pad World Positions

During grasp phase:
```
Static finger box pad (gripper): world pos = [0.2265, 0.0064, 0.0186]
Moving finger box pad:           world pos = [0.2817, 0.0006, 0.0231]
Cube position:                              [0.2532, -0.0009, 0.0269]
```

- Cube Z center: 0.027m
- Static pad Z: 0.019m (8mm below cube center)
- Moving pad Z: 0.023m (4mm below cube center)

The pads are positioned too low to contact the cube sides at center height.

### 4. Mesh Collision Dominates

The gripper has multiple geoms per link:
```
Geoms on gripper link:
  Geom 24: type=<MESH>, contype=1
  Geom 25: type=<MESH>, contype=1
  Geom 26: type=<MESH>, contype=1
  Geom 27: type=<MESH>, contype=1
  Geom 28: type=<BOX>, contype=1  # Finger pad

Geoms on moving_jaw link:
  Geom 29: type=<MESH>, contype=14
  Geom 30: type=<MESH>, contype=14
  Geom 31: type=<BOX>, contype=14  # Finger pad
```

The mesh geoms make contact before/instead of the box pads.

## Genesis Collision System (from documentation)

### Collision Detection Pipeline
1. **Broad Phase**: Sweep-and-Prune with AABB, bitmask filtering (contype/conaffinity)
2. **Narrow Phase**: GJK/MPR for primitives, SDF for meshes

### Key Settings
- `contype`/`conaffinity`: Bitmask collision filtering
- Friction: Uses MAX of both colliding geom frictions
- `use_gjk_collision`: Enable GJK for convex pairs (in RigidOptions)

### Contact Query API
```python
contacts = entity.get_contacts()
# Returns dict with:
#   'valid_mask': (num_envs, max_contacts) bool tensor
#   'geom_a', 'geom_b': (num_envs, max_contacts) int tensors
#   'penetration', 'normal', 'pos', 'friction'
```

## Attempted Solutions

### 1. Runtime contype Modification (Failed)
```python
geom._contype = 0  # After scene.build()
```
Does not take effect - Genesis compiles collision data at build time.

### 2. MJCF Visual Class Modification
Changed `class="visual"` from `contype="1"` to `contype="0"` - no effect on grasp stability since the issue is box positioning, not mesh interference.

## Root Cause Analysis

The finger pad boxes are NOT making contact because:

1. **Position Mismatch**: The boxes are at fingertip height in local frame, but when transformed to world coordinates during grasp, they're below the cube center

2. **Box Size Too Small**: 2.5mm cubes may not protrude enough beyond mesh surfaces in Genesis's collision detection

3. **Possible Frame Difference**: Genesis may interpret MJCF local positions differently than MuJoCo

## Next Steps

1. **Verify box protrusion**: Check if boxes actually protrude beyond mesh in world space during closed grip
2. **Increase box size**: Try larger boxes (5mm+) to ensure they dominate contact
3. **Adjust box positions**: Tune local positions to ensure proper world-space contact
4. **Check Genesis friction model**: May need different friction values than MuJoCo

## Video Output

Test generates: `tests/grasp_stability_test.mp4` at 60fps (2x speed)

Current result: Cube lifts briefly (~4.6cm at step 60) then slips and falls back to table.

## Sources

- [Genesis Collision Documentation](https://genesis-world.readthedocs.io/en/latest/user_guide/advanced_topics/collision_contacts_forces.html)
- [MuJoCo Issue #239 - Mesh collision instability](https://github.com/google-deepmind/mujoco/issues/239)
- devlogs/021_finger_pad_collision_fix.md (MuJoCo solution)
