# Per-Finger Reach Reward (v17 Candidate)

## Problem

Current reward structure gives no gradient for single-finger contact:
- Agent touches cube with one finger
- No reward signal to guide second finger toward cube
- Results in side-pressing behavior instead of proper grasp

### Why Standard Rewards Work Elsewhere But Not Here

Simpler reward structures (like robosuite) work because:

1. **Symmetric parallel-jaw grippers** - Both fingers move equally toward object. Reaching = both fingers approaching. With SO-101's asymmetric gripper, reaching only brings the static finger close.

2. **State-based RL** - Robosuite typically uses perfect proprioception. Closing the gripper is trivial action discovery. With image-based RL, the agent must learn that "this pixel pattern" means "close now."

3. **Forgiving geometry** - Parallel jaws with large pads make grasping easy once positioned. SO-101's asymmetric design requires precise positioning AND active closing.

**Core insight**: Standard `reach_reward = 1 - tanh(gripper_to_cube)` assumes gripper-to-cube distance correlates with "both fingers approaching." For SO-101, that's false - the static finger IS the gripper frame, so reach reward saturates without the moving finger doing anything.

### Why State-Based RL Worked (Devlog 029)

State-based RL with v11 reward achieved 100% success on the same task. Same reward structure, different outcome:

| Aspect | State-based | Image-based |
|--------|-------------|-------------|
| Gripper-to-cube | Direct number | Must infer from pixels |
| Gripper state | Direct value (0-1) | Must infer from visual |
| Credit assignment | Immediate | Delayed through CNN |
| Random grasp discovery | High probability | Very low probability |

The state-based agent can directly observe:
- Exact gripper position relative to cube
- Gripper open/close state as a number
- The relationship: `gripper_state < 0.25 + near_cube = is_grasping`

With images, the agent must learn:
- Visual pattern recognition for "gripper near cube"
- Visual pattern for "gripper closing"
- Temporal relationship between close action and grasp outcome

By the time the image-based agent learns these visual mappings, it has already saturated reach reward with the open-gripper hovering policy. There's no gradient to discover the grasp reward.

v17's moving finger reach explicitly encodes the "close when near" gradient that state-based RL got for free through direct observation.

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
2. **Cap on `is_closed`**: Caps when gripper_state < 0.25 (not on contact detection)
3. **Strong gradient**: ~0.4 reward difference incentivizes closing gripper
4. **Proximity gate**: Only applies moving finger reach when close to cube

## Proximity Gating

### Problem

Moving finger reach can backfire when the cube is far away:
- If gripper is far from cube and rotated weirdly, the moving finger might be closer to cube than the gripper TCP
- This creates a perverse incentive to OPEN the gripper (moving finger stays put while TCP moves away)
- Net effect: agent learns to keep gripper open when far from cube

### Solution

Only compute moving finger reach when already close to the cube:

```python
reach_threshold = 0.7  # gripper_reach at ~3cm from cube

if gripper_reach < reach_threshold:
    # Far from cube: just use gripper reach, no closing incentive yet
    reach_reward = gripper_reach
else:
    # Close to cube: blend in moving finger reach
    if is_closed:
        moving_reach = 1.0
    else:
        moving_finger_pos = self._get_moving_finger_pos()
        moving_to_cube = np.linalg.norm(moving_finger_pos - cube_pos)
        moving_reach = 1.0 - np.tanh(10.0 * moving_to_cube)

    reach_reward = (gripper_reach + moving_reach) * 0.5
```

### Rationale

1. **When far from cube** (`gripper_reach < 0.7`):
   - Agent should focus on approaching the cube
   - No incentive to close or open gripper prematurely
   - Standard reach reward provides smooth gradient

2. **When close to cube** (`gripper_reach >= 0.7`):
   - Agent is positioned for grasp attempt
   - Moving finger reach provides gradient to close gripper
   - Capping at `is_closed` gives exploration room to discover grasp bonus

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

## Reward Structure Summary

```
v17 reward = reach_reward + grasp_bonus + lift_bonus

reach_reward:
  if gripper_reach < 0.7:
    reach_reward = gripper_reach  # Standard reach only
  else:
    reach_reward = (gripper_reach + moving_reach) / 2  # Blend both

grasp_bonus: 1.5 (when is_grasping=True)
lift_bonus: tanh(10 * (cube_z - 0.03)) when grasping
```
