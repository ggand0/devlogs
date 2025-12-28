# Devlog 020: Top-Down Pick Implementation

## Reference

Adapted from ECE4560 SO-101 Assignment 8: https://maegantucker.com/ECE4560/assignment8-so101/

## Implementation

### 5-Step Sequence

1. **Move above block** - Position gripperframe 30mm above cube with gripper open
2. **Move down to block** - Lower to grasp height with gripper open
3. **Close gripper** - Gradual close with contact detection, then tighten
4. **Lift** - Raise cube with reduced IK gain to minimize wobble
5. **Hold** - Maintain position to verify stable grasp

### Key Offsets

```python
grasp_z_offset = 0.005      # 5mm above cube center
finger_width_offset = -0.015 # Y offset to center asymmetric gripper
height_offset = 0.03         # 30mm clearance above cube
```

### Critical Decisions

**gripperframe vs graspframe**: Use `gripperframe` site for IK targeting - it's at the fingertips. `graspframe` is 6cm behind (see devlog 015).

**Wrist locking**: Lock joints 3 and 4 during IK to maintain top-down orientation:
- `wrist_flex = pi/2` (pointing down)
- `wrist_roll = pi/2` (fingers along Y axis)

**Y offset**: The gripper is asymmetric (static vs moving finger). Offset target by -15mm in Y to center the grip on the cube.

**Contact-based grip**: Close gradually until `is_grasping()` returns True (both fingers touching cube), then tighten by 0.4 more.

**Reduced lift gain**: Use `gain=0.3` during lift/hold (vs 0.5 for approach) to reduce IK oscillation that causes wobble.

### Grasp Detection

```python
def is_grasping():
    contacts = get_contacts()
    has_static = any(g in contacts for g in [27, 28])  # static finger geoms
    has_moving = any(g in contacts for g in [29, 30])  # moving finger geoms
    return has_static and has_moving
```

### Known Issues

- Moving finger penetrates cube slightly, causing tilt
- Some residual wobble during lift
- Hardcoded finger geom IDs (27, 28, 29, 30)

### Files

- `tests/test_topdown_pick.py` - Main implementation
- `src/controllers/ik_controller.py` - IK controller used for positioning
