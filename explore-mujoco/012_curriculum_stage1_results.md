# Devlog 012: Curriculum Stage 1 Training Results

## Experiment Overview

Two runs tested Curriculum Stage 1 (cube starts in closed gripper at lift height):

| Run | Reward Version | Path |
|-----|----------------|------|
| V7 | v7 (V1 + push-down penalty) | `runs/lift_curriculum_s1_v7/20251219_034126/` |
| V8 | v8 (V7 + drop penalty) | `runs/lift_curriculum_s1/20251219_052903/` |

Both used:
- `curriculum_stage: 1` - Cube spawns already grasped at lift height (0.08m)
- 500k timesteps, SAC with lr=3e-4
- `max_episode_steps: 100`, `hold_steps: 10`

## Results Summary

### Success Rate: 0% for Both

Neither agent achieved the success condition (hold cube above 0.08m for 10 steps).

**V7 Final Eval:**
```
Episode rewards: [25.88, 51.64, 31.09, 39.61, 90.07]
Episode lengths: [100, 100, 100, 100, 100] (all truncated)
Successes: [False, False, False, False, False]
Peak mean reward: 59.25 at 350k steps
```

**V8 Final Eval:**
```
Episode rewards: [37.91, 45.64, 24.20, 45.25, 22.12]
Episode lengths: [100, 100, 100, 100, 100] (all truncated)
Successes: [False, False, False, False, False]
Peak mean reward: 75.03 at 160k steps
```

## The Core Problem: IK Cannot Grasp

The curriculum Stage 1 reset uses the IK controller to position the cube in the gripper:

```python
def _reset_with_cube_in_gripper(self, cube_qpos_addr: int, lift_height: float):
    # Step 1: Move gripper above cube
    # Step 2: Lower onto cube
    # Step 3: Close gripper
    # Step 4: Lift to target height
```

**This fails because the IK controller only does position control, not orientation control.**

The gripper approaches the cube from the side (horizontal orientation) rather than from above (vertical/top-down orientation). When the gripper closes from the side at table level, it cannot achieve a stable two-sided grasp.

### Evidence from Manual Testing

Extensive testing in `test_grasp_frames.py` confirmed:

1. Side approach (IK default): Gripper pushes cube sideways, cannot achieve stable grasp
2. Top-down approach (manual joint control): Achieves two-sided grasp with cube lift

Working top-down configuration found:
```python
# Phase 1: Rotate wrist to point down (joint 3: 0.0 → 1.65)
# Phase 2: Lower elbow to cube level (joint 2: -1.0 → -0.3)
# Close gripper
# Result: contacts=[0, 0, 28, 30] (both fingers), cube lifted z=0.015→0.018
```

## Why Curriculum Stage 1 Fails to Initialize Correctly

The IK-based reset doesn't produce a stable grasp. The cube either:
1. Falls out immediately when the episode starts
2. Is held by a single finger contact (unstable)
3. Is pushed away before grasping can occur

Since Stage 1 assumes the agent starts with a grasped cube, but the reset never achieves a real grasp, the agent can't learn to "hold" something it never had.

## Reward Comparison: V7 vs V8

V8 added a drop penalty on top of V7:
```python
# V8: Penalize losing grasp after having it
if was_grasping and not is_grasping:
    reward -= 2.0
```

The drop penalty had no effect because the agent never had a grasp to begin with.

## Next Steps

1. **Fix the grasp initialization** - Either:
   - Add orientation control to IK controller
   - Use the working manual joint sequence for reset
   - Pre-compute joint positions for a top-down grasp pose

2. **Alternative curriculum approach**:
   - Skip Stage 1/2 (cube-in-gripper stages)
   - Start with Stage 3 (gripper near cube, open)
   - Let agent learn to grasp from scratch with stiff physics

3. **Consider the test_grasp_frames.py findings**:
   - Two-phase lowering (wrist rotate, then elbow lower) works
   - Use this sequence in the reset function

## Files

- `runs/lift_curriculum_s1_v7/20251219_034126/` - V7 training run
- `runs/lift_curriculum_s1/20251219_052903/` - V8 training run
- `test_grasp_frames.py` - Manual grasp configuration testing
- `docs/004_top-down-grasp-configuration.md` - Working grasp config
