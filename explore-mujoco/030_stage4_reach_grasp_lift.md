# Devlog 030: Stage 4 - Reach + Grasp + Lift

## Overview

Implementation of curriculum stage 4, where the gripper starts further from the cube and must reach before grasping.

## Progression

| Stage | Initial State | Task |
|-------|---------------|------|
| 3 ✅ | Gripper ~3cm above cube | Grasp + Lift + Hold |
| **4** | Gripper 8-12cm away in XY | Reach + Grasp + Lift + Hold |
| 5 | (future) | Pick + Transport |
| 6 | (future) | Pick + Place |

## Implementation

### Environment Changes (`src/envs/lift_cube.py`)

Added `_reset_gripper_far_from_cube()` function:

```python
def _reset_gripper_far_from_cube(self, cube_qpos_addr: int):
    """Reset with gripper positioned further from cube - must reach first.

    Stage 4: Gripper starts ~8-12cm away from cube in XY plane,
    at lift height. Agent must reach, descend, grasp, and lift.
    """
```

Key parameters:
- **Distance**: 8-12cm from cube (randomized)
- **Direction**: Random angle (0 to 2π)
- **Height**: Starts at `lift_height + 0.02` (10cm above table)
- **Gripper state**: Open (0.3)

The gripper position is clamped to workspace bounds:
- X: 0.15 to 0.45
- Y: -0.25 to 0.25

### Configuration (`configs/curriculum_stage4.yaml`)

```yaml
env:
  curriculum_stage: 4       # Gripper far from cube
  max_episode_steps: 400    # More time for reach + grasp + lift
  hold_steps: 150           # 3 second hold
  reward_version: "v11"     # Same reward as stage 3
  lock_wrist: true

training:
  timesteps: 2000000        # 2M steps (2x stage 3)
```

### Reward Structure

Uses same v11 reward as stage 3:
- Reach reward: `1.0 - tanh(10 * gripper_to_cube)`
- Grasp bonus: +0.25
- Continuous lift reward when grasping
- Binary lift bonus at z > 0.02
- Target height bonus at z > 0.08
- Action rate penalty when z > 0.06
- Success bonus: +10.0

The reach reward naturally guides the agent toward the cube from any starting position.

## Expected Behavior

1. Move gripper horizontally toward cube
2. Descend to grasp height
3. Close gripper
4. Lift to target height
5. Hold for 150 steps

## Training

```bash
PYTHONPATH=. uv run python train_lift.py --config configs/curriculum_stage4.yaml
```

Expected training time: ~8 hours on RTX 4090 (2M steps)

## Also Added: Pick-and-Place Foundation

For future stages 5/6, added:

### Environment
- `place_target` parameter: (x, y) target location
- Observation space extended with target position (+3 dims)
- `_reward_v12`: Transport and placement rewards
- Success condition: cube at target, on ground, gripper released

### Configuration (`configs/pick_place.yaml`)
- `place_target: [0.35, 0.10]` (~15cm from start)
- `reward_version: "v12"`
- Agent controls gripper for release

This will be used after stage 4 succeeds.

## Files Changed

- `src/envs/lift_cube.py` - Added stage 4 reset, place_target support, v12 reward
- `configs/curriculum_stage4.yaml` - Stage 4 configuration
- `configs/pick_place.yaml` - Future pick-and-place configuration
- `train_lift.py` - Added place_target parsing
- `eval_cartesian.py` - Added --run flag, place task logging
