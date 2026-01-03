# V16 Reward: Increased Grasp Bonus

## Problem

After the camera calibration change (devlog 043), training got stuck at a local optimum:
- Agent positions gripper near cube (maximizing reach reward ~0.9)
- Keeps gripper wide open (state 1.2-1.78)
- Never attempts to grasp

V15 tried to fix this with a gripper-open penalty, but the root cause is that the grasp bonus (+0.25) is too weak compared to reach reward (~1.0).

## Analysis

### Current Reward Math (v14/v15)

| Behavior | Reach | Grasp | Total |
|----------|-------|-------|-------|
| Hovering near cube | ~0.9 | 0 | **~0.9** |
| Grasping | ~0.9 | +0.25 | **~1.15** |
| **Delta** | | | **+0.25** |

The +0.25 delta isn't worth the exploration cost of learning to close the gripper.

### V16 Reward Math (grasp bonus = 1.5)

| Behavior | Reach | Grasp | Total |
|----------|-------|-------|-------|
| Hovering near cube | ~0.9 | 0 | **~0.9** |
| Grasping | ~0.9 | +1.5 | **~2.4** |
| **Delta** | | | **+1.5** |

Now grasping gives ~2.7x the reward of hovering.

### Full Reward Gradient (v16)

| State | Reach | Grasp | Lift Progress | Binary Lift | Target | Total |
|-------|-------|-------|---------------|-------------|--------|-------|
| Hover near cube | 0.9 | 0 | 0 | 0 | 0 | **0.9** |
| Grasp at ground (z=0.015) | 0.9 | 1.5 | 0 | 0 | 0 | **2.4** |
| Grasp at z=0.02 | 0.9 | 1.5 | 0.15 | 1.0 | 0 | **3.55** |
| Grasp at z=0.08 (target) | 0.9 | 1.5 | 2.0 | 1.0 | 1.0 | **6.4** |

### Gradient Analysis

| Transition | Delta |
|------------|-------|
| Hover → Grasp | **+1.5** |
| Grasp → Lift (0.08) | **+4.0** |

The lift gradient (+4.0) is bigger than the grasp bonus (+1.5), so the agent still has strong incentive to lift after grasping. The grasp bonus shifts everything up uniformly without compressing the lift gradient.

## Design Decision: Remove Gripper-Open Penalty

V16 removes the gripper-open penalty from v15. Reasons:

1. **Cleaner experiment** - If v16 works, we know it's the grasp bonus alone
2. **Positive > Negative** - The 1.5 grasp bonus directly rewards what we want. The penalty was a workaround for the weak +0.25 bonus
3. **Potential interference** - Agent might learn to close gripper at wrong times just to avoid penalty
4. **Simpler debugging** - If v16 fails, we know grasp bonus alone isn't enough

## Implementation

### V16 Reward Function

Based on v14 with grasp bonus changed from 0.25 to 1.5:

```python
def _reward_v16(self, info, was_grasping=False, action=None):
    reward = 0.0

    # Reach reward (unchanged)
    reach_reward = 1.0 - np.tanh(10.0 * gripper_to_cube)
    reward += reach_reward

    # Push-down penalty (unchanged)
    if cube_z < 0.01:
        reward -= (0.01 - cube_z) * 50.0

    # Drop penalty (unchanged)
    if was_grasping and not is_grasping:
        reward -= 2.0

    # Grasp bonus (INCREASED from 0.25 to 1.5)
    if is_grasping:
        reward += 1.5

        # Lift rewards (unchanged)
        lift_progress = max(0, cube_z - 0.015) / (lift_height - 0.015)
        reward += lift_progress * 2.0

        if cube_z > 0.02:
            reward += 1.0

    # Target height bonus (unchanged)
    if cube_z > lift_height:
        reward += 1.0

    # Action penalty during hold (unchanged)
    # Success bonus (unchanged)

    return reward
```

### Files Changed

- `src/envs/lift_cube.py` - Added `_reward_v16()` method
- `configs/drqv2_lift_s3_v16.yaml` - New config with `reward_version: v16`

## Training Command

```bash
MUJOCO_GL=egl uv run python src/training/train_image_rl.py --config configs/drqv2_lift_s3_v16.yaml
```

## Training Runs

| Version | Directory | Notes |
|---------|-----------|-------|
| v15 | `runs/image_rl/20260102_234617/` | Terminated at 800k, stuck at local optimum |
| v16 | `runs/image_rl/20260103_030013/` | Current run |

### V15 Results (terminated)
- Saturated at `ep_rew_mean ~150` since 300k steps
- Agent learned to briefly close/open gripper to game the penalty
- Single-finger movement persisted
- No grasping behavior achieved

## What to Watch For

| Step | Metric | Good Sign |
|------|--------|-----------|
| Early (50-100k) | `ep_rew_mean` | Higher than v14/v15 (~150+ vs ~115) |
| Mid (200-500k) | `is_grasping` in debug logs | Agent attempting to close gripper |
| Later (500k+) | `success_rate` | Any non-zero success |

## Backup Plans if V16 Fails

1. **Add gripper-close reward** - Positive shaping when closing near cube:
   ```python
   if gripper_to_cube < 0.05:
       close_reward = 0.3 * max(0, 0.3 - gripper_state)
       reward += close_reward
   ```

2. **Stage 2.5 curriculum** - Cube loosely in gripper, just needs to close and lift

3. **Two-finger contact reward** - Use MuJoCo contacts to detect both fingertips touching cube

4. **Train on old camera first** - Bootstrap grasping behavior, then transfer to new camera

## Related Devlogs

- [043: Camera FOV Calibration](./043_camera_fov_calibration.md) - Camera change that broke v13
- [044: V14 Reward Investigation](./044_v14_reward_investigation.md) - Analysis of v14 failure
- [045: V15 Gripper-Open Penalty](./045_v15_gripper_open_penalty.md) - Penalty approach (now superseded)
