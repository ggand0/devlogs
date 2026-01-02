# Devlog 010: V5-V7 Training Summary

## Key Finding

**V1's "grasping" was a physics exploit.** The soft contact parameters (`solref="0.02 1"`) allowed the gripper to clip through the cube, achieving (True, True) contact artificially. Once we fixed the physics to stiff contacts (`solref="0.001 1"`), true grasping has never been achieved.

## Results Summary

| Version | Physics | Reward Changes | Contacts | Gripper | Cube Z | Behavior |
|---------|---------|---------------|----------|---------|--------|----------|
| **V1** | Soft | Reach + grasp(+0.25) + binary lift | (True, True) | Closed (-0.16) | 0.007 | "Grasped" via clipping, pushed down |
| **V3** | Soft | V1 + continuous lift | Mixed | Mixed | 0.008-0.013 | Destabilized, explored then hovered |
| **V4** | Soft | V3 but grasp only when elevated | (True, False) | Open (0.76-1.22) | 0.007 | Never closes gripper |
| **V5** | Stiff | V3 + push-down penalty | (True, False) | Half-open (0.30-0.85) | 0.014 | Nudge exploit - tilts cube |
| **V6** | Stiff | V5 - lift_baseline | (False, False) | Open | 0.010 | Safe hover far away (~14cm) |
| **V7** | Stiff | V1 + push-down penalty | (True, False) | Open (0.77-1.30) | 0.010 | One-sided contact, never closes |

## V7 Training Run

```
Directory: runs/lift_cube/20251216_150046/
Reward: V7 (V1 + push-down penalty)
Physics: Stiff
Timesteps: 500k
Training time: ~1h 53min
```

### V7 Reward Structure

```python
reward = 0.0
reward += 1.0 - np.tanh(10.0 * gripper_to_cube)  # Reach

if cube_z < 0.01:
    reward -= (0.01 - cube_z) * 50.0  # Push-down penalty

if is_grasping:
    reward += 0.25  # Grasp bonus (same as V1)

if cube_z > 0.02:
    reward += 1.0  # Binary lift

if is_success:
    reward += 10.0
```

### V7 Evaluation Results

| Episode | Gripper | Distance | Cube Z | Contacts | Behavior |
|---------|---------|----------|--------|----------|----------|
| 1 | 0.78-0.89 (open) | 0.015m | 0.010-0.011 | (True, False) | Press with open gripper |
| 2 | 0.83-0.89 (open) | 0.015m | 0.010 | (True, False) | Same |
| 3 | 1.12-1.30 (open) | 0.015-0.017m | 0.010 | (True, False) | Same |
| 4 | 1.12-1.26 (open) | 0.015-0.016m | 0.010 | (True, False) | Same |
| 5 | 0.77-0.80 (open) | 0.015m | 0.010 | (True, False) | Same |

- **Mean reward**: 163.22 +/- 0.98
- **Success rate**: 0%

## Core Problem: Sparse Grasp Reward

The grasp reward requires **3 conditions simultaneously**:
1. Gripper closed (< 0.25)
2. Static gripper contact with cube (True)
3. Moving jaw contact with cube (True)

With stiff physics, random exploration rarely discovers this combination. The agent finds it easier to:
- Press with open gripper (get reach reward, avoid risk)
- Hover safely (saturated reach reward ~0.9)

## Why V1 Appeared to Work

V1's soft physics allowed mesh clipping:
- Gripper tip could penetrate cube geometry
- This triggered both contact flags (True, True) without true pinching
- Agent learned to "grasp" by clipping, then push down

The stiff physics fix closed this exploit but revealed the underlying problem: the grasp reward is too sparse for exploration to discover naturally.

## Next Steps

Options to address sparse grasp reward:
1. **Partial contact reward** - Give small bonus for any cube contact + closing gripper
2. **Curriculum learning** - Start with cube already in gripper, learn to lift first
3. **Demonstration data** - Show the agent what grasping looks like
4. **Soften grasp requirement** - Reward closing gripper when near cube, regardless of contacts

## V7 Resumed Training (500k → 1M)

```
Directory: runs/lift_cube/20251216_170544_resumed/
Resumed from: runs/lift_cube/20251216_150046/ (V7 @ 500k)
Additional steps: 500k (total: 1M)
```

### V7 @ 1M Evaluation Results

| Episode | Gripper | Distance | Cube Z | Contacts | Behavior |
|---------|---------|----------|--------|----------|----------|
| 1 | 0.27-1.52 | 0.015m | 0.010 | (True, False) | Fidgets with bottom of gripper |
| 2 | 0.30-1.20 | 0.015m | 0.010 | (True, False) | Same fidgeting |
| 3 | 0.35-1.40 | 0.015m | 0.010 | (True, False) | Same |
| 4 | 0.28-1.35 | 0.015m | 0.010 | (True, False) | Same |
| 5 | 0.32-1.45 | 0.015m | 0.010 | (True, False) | Same |

- **Mean reward**: 159.26 ± 2.95
- **Success rate**: 0%

### Behavior Analysis

The agent "fidgets around the cube with the bottom part of gripper" - it's learned to make contact with the static gripper side but never achieves both-sided contact needed for grasping. The gripper oscillates between partially open (0.27) and fully open (1.52), showing some exploration of closing but never committing.

**Key observation**: This fidgeting behavior is more engaged than pure hovering, suggesting the agent is closer to discovering grasping but lacks the final signal to close the gripper while contacting both sides.

### Comparison: 500k vs 1M

| Metric | V7 @ 500k | V7 @ 1M |
|--------|-----------|---------|
| Mean reward | 163.22 ± 0.98 | 159.26 ± 2.95 |
| Success rate | 0% | 0% |
| Gripper range | 0.77-1.30 | 0.27-1.52 |
| Contacts | (True, False) | (True, False) |
| Behavior | Press with open gripper | Fidgets, more gripper movement |

The extended training slightly decreased reward but increased behavioral variability - the gripper now explores closing more (0.27 vs 0.77 minimum). Still insufficient to achieve actual grasping.

## Files

- `envs/lift_cube.py` - V7 reward function
- `runs/lift_cube/20251216_150046/` - V7 training run (500k)
- `runs/lift_cube/20251216_170544_resumed/` - V7 resumed training (1M)
- `runs/lift_cube/20251216_170544_resumed/v7_1M_eval_combined.mp4` - Evaluation video
- `runs/lift_cube/20251216_170544_resumed/v7_full_curve.png` - Learning curve
