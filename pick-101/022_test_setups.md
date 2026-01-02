# Devlog 022: Test Script Configurations

## Summary

Documented the three distinct test configurations for the SO-101 arm, each with its own robot model and scene file to preserve reproducibility as the main model evolves.

## The Problem

As `so101_new_calib.xml` evolved with improvements (finger pads, fingertip sites, friction tuning), old test scripts broke because they relied on hardcoded geom IDs and specific physics settings. Creating version-specific XML files allows each script to use the exact configuration it was designed for.

## Three Test Configurations

### 1. Top-Down Pick (Current)

**Files:**
- Robot: `so101_new_calib.xml`
- Scene: `lift_cube_scene.xml`
- Script: `tests/test_topdown_pick.py`

**Features:**
- Finger pad collision boxes for stable multi-contact grasping
- Fingertip sites for precise targeting
- Elliptic cone friction (`cone="elliptic"`, `impratio="10"`)
- Higher friction coefficients for slip prevention

**Geom IDs:**
| ID | Part |
|----|------|
| 27-28 | Static finger mesh |
| 29-30 | Moving jaw mesh |
| 31-32 | Finger pad boxes |
| 33 | Cube |

**Command:**
```bash
PYTHONPATH=. uv run python tests/test_topdown_pick.py
```

### 2. IK Grasp (Legacy)

**Files:**
- Robot: `so101_ik_grasp.xml` (from commit bec853b)
- Scene: `lift_cube_scene_ik_grasp.xml`
- Scripts: `tests/test_ik_grasp.py`, `tests/test_ik_grasp_video.py`

**Features:**
- No finger pads (mesh-only collision)
- No fingertip sites
- Default MuJoCo friction (pyramidal cone)
- Collision exclusions between gripper parts

**Geom IDs:**
| ID | Part |
|----|------|
| 27-28 | Static finger mesh |
| 29-30 | Moving jaw mesh |
| 31 | Cube |

**Targeting Method:**
Uses finger midpoint targeting - calculates offset from TCP to finger midpoint and adjusts IK target accordingly:
```python
def get_finger_mid():
    f28 = data.geom_xpos[28]
    f30 = data.geom_xpos[30]
    return (f28 + f30) / 2

def get_tcp_to_finger_offset():
    tcp = ik.get_ee_position()
    finger_mid = get_finger_mid()
    return tcp - finger_mid

# Target: position TCP so fingers end up at cube
finger_target = np.array([cube_x, cube_y, cube_z])
tcp_target = finger_target + get_tcp_to_finger_offset()
```

**Wrist Configuration:**
- `wrist_flex = 1.65` (pointing down)
- `wrist_roll = π/2` (horizontal fingers)
- Locked joints [3, 4] during IK solve

**Command:**
```bash
PYTHONPATH=. uv run python tests/test_ik_grasp.py          # Interactive
PYTHONPATH=. uv run python tests/test_ik_grasp_video.py   # Saves video
```

**Output:**
Video saved to `runs/ik_grasp_test/ik_grasp_finger_target.mp4`

### 3. Horizontal Grasp (Experimental)

**Files:**
- Robot: `so101_horizontal_grasp.xml` (from commit b686a9e)
- Scene: `lift_cube_scene_horizontal.xml`
- Script: `tests/test_horizontal_grasp.py`

**Features:**
- Horizontal approach (cube at elevated height)
- Elliptic cone friction enabled
- No finger pads

**Geom IDs:** Same as IK Grasp (27-30 for fingers, 31 for cube)

**Command:**
```bash
PYTHONPATH=. uv run python tests/test_horizontal_grasp.py
```

## Model Evolution Timeline

| Commit | Changes | Used By |
|--------|---------|---------|
| bec853b | Collision exclusions added | test_ik_grasp*.py |
| 039169f | Gripperframe Y-centering | - |
| b686a9e | Elliptic cone friction | test_horizontal_grasp.py |
| e74f85d | Fingertip sites added | - |
| 0b8790d | Finger pad collision boxes | - |
| fafd639 | Finger pad position refinement | test_topdown_pick.py |

## File Relationships

```
models/so101/
├── so101_new_calib.xml       # Current (finger pads, fingertip sites)
├── so101_ik_grasp.xml        # Legacy (no pads)
├── so101_horizontal_grasp.xml # Experimental
├── lift_cube.xml             # Current scene (elliptic friction)
├── lift_cube_ik_grasp.xml    # Legacy scene (default friction)
├── lift_cube_horizontal.xml  # Horizontal scene
└── assets/                   # STL mesh files (Git LFS)
```

## Notes

- Old scripts cannot reliably lift the cube due to slip issues - this is expected behavior and why finger pads were added
- Geom IDs are model-specific and will differ between configurations
- Each scene includes only the robot model it was designed for
