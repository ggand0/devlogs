# Per-Finger Reach Reward (v17 Candidate)

## Problem

Current reward structure gives no gradient for single-finger contact:
- Agent touches cube with one finger
- No reward signal to guide second finger toward cube
- Results in side-pressing behavior instead of proper grasp

## Proposed Solution: Moving Finger Reach Reward

Add a reach reward for the moving finger that caps on contact. The static finger is part of the gripper frame (same as TCP), so we keep standard gripper-to-cube reach for that:

```python
def _reward_v17(self, info, was_grasping, action):
    """Moving finger reach reward with contact cap."""
    gripper_to_cube = info["gripper_to_cube"]
    has_moving_contact = info["has_jaw_contact"]

    # Standard gripper reach (static finger is part of gripper frame)
    gripper_reach = 1.0 - np.tanh(10.0 * gripper_to_cube)

    # Moving finger reach - caps at 1.0 on contact
    if has_moving_contact:
        moving_reach = 1.0  # maxed out, no more gradient
    else:
        moving_finger_pos = self._get_moving_finger_pos()
        moving_to_cube = np.linalg.norm(moving_finger_pos - cube_pos)
        moving_reach = 1 - np.tanh(10 * moving_to_cube)

    # Combined reach reward (0 to 1)
    reach_reward = (gripper_reach + moving_reach) * 0.5

    # Rest of reward same as v13...
```

## Reward Gradient

| State | Gripper Reach | Moving Reach | Combined |
|-------|---------------|--------------|----------|
| Near cube, gripper open | ~0.83 | ~0.25 | ~0.54 |
| Near cube, gripper closed | ~0.89 | 1.0 (capped) | ~0.94 |
| Difference | | | **+0.4/step** |

## Key Design Decisions

1. **Only moving finger**: Static finger is part of gripper frame (same as TCP)
2. **Cap on contact**: Prevents agent from trying to push finger through cube
3. **Strong gradient**: ~0.4 reward difference incentivizes closing gripper

## Implementation Requirements

1. Add finger position getters to environment:
   - `_get_static_finger_pos()` - position of static gripper pad
   - `_get_moving_finger_pos()` - position of moving jaw pad

2. Finger positions can be computed from:
   - Gripper TCP position
   - Gripper state (opening width)
   - Known gripper geometry

## Status: Implemented

Bootstrap training failed at 600k steps (same single-finger open-gripper behavior).
v17 has been implemented and tested.

### Files Changed
- `src/envs/lift_cube.py`: Added `_get_static_finger_pos()`, `_get_moving_finger_pos()`, `_reward_v17()`
- `configs/drqv2_lift_s3_v17.yaml`: Stage 3 config with v17 reward

### Test Results (Updated)

Only moving finger gets per-finger reach (static finger is part of gripper frame):

```
After descend (open gripper):
  Gripper reach: 0.833
  Moving reach: 0.245 (far because gripper open)
  Combined: 0.539

After close:
  Gripper reach: 0.889
  Moving reach: 1.000 (capped on contact)
  Combined: 0.944
```

**Key insight**: ~0.4 reward difference per step between open and closed gripper when near cube. This gives strong gradient to close the gripper.

## Alternative: Simpler Version

If getting finger positions is complex, use contact-based bonus instead:

```python
# Simpler: just add per-finger contact bonus
contact_bonus = 0.0
if has_static_contact:
    contact_bonus += 0.15
if has_moving_contact:
    contact_bonus += 0.15
# Full grasp still gets additional is_grasping bonus
```

This gives gradient but no smooth approach reward.
