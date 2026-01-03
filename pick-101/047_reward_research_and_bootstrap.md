# Reward Research for Image-Based RL Grasping

## Problem

After camera calibration (devlog 043), v14-v16 rewards all fail to learn grasping:
- Agent keeps gripper wide open (mean 1.2-1.7, threshold < 0.25)
- Only single-finger contact, never two-finger grasp
- No gradient toward closing gripper

## Research Summary

### 1. Staged Rewards (Robosuite Standard)

[Robosuite pick_place.py](https://github.com/ARISE-Initiative/robosuite/blob/master/robosuite/environments/manipulation/pick_place.py) uses:

| Stage | Range | Formula |
|-------|-------|---------|
| Reaching | [0, 0.1] | `(1 - tanh(10 * dist)) * 0.1` |
| Grasping | {0, 0.35} | Binary contact detection |
| Lifting | [0.35, 0.5] | Only if grasping; proportional to height |
| Hovering | [0.5, 0.7] | Only if lifted; toward target |

Key insight: Reaching reward provides smooth gradient toward object.

### 2. QT-Opt (Google, 96% Success)

[QT-Opt paper](https://arxiv.org/abs/1806.10293) achieves 96% grasp success with **sparse rewards**:

**Reward:**
- `+1` if object in gripper at height (terminal)
- `-0.05` per step (encourages speed)
- `0` otherwise

**Exploration Solution:**
> "During the early stages of training, a policy that takes random actions would achieve reward too rarely to learn from, since the grasping task is a multi-stage problem and reward is sparse."

**Scripted Policy Bootstrap:**
- Collect initial data using scripted policy with 15-30% success rate
- Scripted policy: random (x,y), lower gripper, close, lift
- This seeds the replay buffer with successful grasps
- RL then improves from this baseline

### 3. Dense2Sparse (2020)

[Dense2Sparse](https://arxiv.org/abs/2003.02740):
- Start with dense rewards for fast initial learning
- Transition to sparse rewards to avoid reward hacking
- Best of both worlds

### 4. Reward Training Wheels (2025)

[RTW](https://arxiv.org/abs/2503.15724):
- Automates auxiliary reward adaptation
- Teacher-student framework
- Removes human bias from reward engineering

## Our Problem Analysis

Current reward structure:
```
Reach reward: 1 - tanh(10 * distance)  → ~0.9 at close range
Grasp bonus: +1.5 (v16)                → requires is_grasping=True
```

`is_grasping` requires:
1. `gripper_state < 0.25` (closed)
2. Both finger pads touching cube

The agent maximizes reach reward (~0.9/step) without ever discovering grasp reward because:
- No gradient toward closing gripper
- Random exploration rarely produces close + two-finger contact simultaneously
- Single-finger contact gives same reach reward as two-finger

## Proposed Solution: Scripted Bootstrap

Following QT-Opt approach:

1. **Create scripted grasp policy** that achieves 15-30% success
2. **Seed replay buffer** with scripted trajectories
3. **Train RL** from this baseline

### Scripted Policy Design

```python
def scripted_grasp(env):
    """Simple scripted policy for bootstrapping."""
    # Phase 1: Descend with open gripper
    for _ in range(20):
        action = [0, 0, -0.5, 1.0]  # down, gripper open
        env.step(action)

    # Phase 2: Close gripper
    for _ in range(10):
        action = [0, 0, 0, -1.0]  # stay, gripper close
        env.step(action)

    # Phase 3: Lift
    for _ in range(30):
        action = [0, 0, 0.5, -1.0]  # up, gripper closed
        env.step(action)
```

With curriculum stage 3 (gripper near cube), this should achieve decent success rate.

### Implementation Plan

1. Create `scripts/collect_scripted_grasps.py`
2. Save trajectories to replay buffer format
3. Modify training to load pre-seeded buffer
4. Train with sparse or current reward

## Alternative: Staged Contact Rewards

If bootstrap doesn't work, add intermediate rewards:

```python
# Reward each finger contact separately
if has_static_contact:
    reward += 0.15
if has_moving_contact:
    reward += 0.15

# Closing reward when near cube
if gripper_to_cube < 0.05:
    close_progress = max(0, 0.4 - gripper_state) / 0.4
    reward += close_progress * 0.2
```

## References

- [QT-Opt: Scalable Deep RL for Vision-Based Manipulation](https://arxiv.org/abs/1806.10293)
- [Robosuite Environments](https://robosuite.ai/docs/modules/environments.html)
- [Dense vs Sparse Visual Rewards Study](https://ar5iv.labs.arxiv.org/html/2108.03222)
- [Reward Training Wheels](https://arxiv.org/abs/2503.15724)
- [Dense2Sparse Reward Shaping](https://arxiv.org/abs/2003.02740)
- [DrQ-v2 GitHub](https://github.com/facebookresearch/drqv2)

## Next Steps

1. Change cube color to red (match IRL setup)
2. Implement scripted grasp collection
3. Test scripted policy success rate
4. If >15%, proceed with bootstrap training
