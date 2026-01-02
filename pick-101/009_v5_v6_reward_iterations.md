# Devlog 009: V5 and V6 Reward Iterations

## V5: Push-Down Penalty

### Changes from V4
- Removed collision boxes (they blocked grasping - see devlog 007)
- Added push-down penalty: `reward -= (0.01 - cube_z) * 50.0` when `cube_z < 0.01`
- Kept lift_baseline reward for any cube elevation

### Training Run
```
Directory: runs/lift_cube/20251216_094258/
Reward: V5 (push-down penalty + lift_baseline)
Timesteps: 500k
Training time: ~1h 52min
```

### Results at 500k

| Episode | Gripper | Distance | Cube Z | Contacts | Behavior |
|---------|---------|----------|--------|----------|----------|
| 1 | 0.30 (half-open) | 0.014m | 0.014 | (True, False) | Nudge cube with static gripper |
| 2 | 0.48 (half-open) | 0.015m | 0.014 | (True, False) | Same nudge pattern |
| 3 | 0.85 (open) | 0.015m | 0.014 | (True, False) | Same nudge pattern |
| 4 | 0.30 (half-open) | 0.014m | 0.014 | (True, False) | Same nudge pattern |
| 5 | 0.77 (open) | 0.016m | 0.014 | (True, False) | Same nudge pattern |

- **Mean reward**: 164.46 +/- 0.69
- **Success rate**: 0%

### Analysis

**Good**: Push-down penalty works - cube stays at z=0.014 instead of being pushed to z=0.007.

**Bad**: Agent found new exploit - nudge cube with one-sided contact to get lift_baseline reward without grasping.

At z=0.014 with one-sided contact:
- No grasp: 0.90 (reach) + 0.04 (lift_baseline) = **0.94/step**
- With grasp: 0.94 + 0.25 + 0.16 = **1.35/step**

The 0.41 difference per step isn't enough to overcome the "safety" of not attempting to grasp.

### V5 Exploit

The agent learned to:
1. Approach cube from side
2. Press against cube with static gripper (one-sided contact)
3. Tilt cube slightly to z=0.014
4. Hold position, collecting lift_baseline reward

This avoids the complexity of closing gripper and achieving two-sided contact.

---

## V6: Remove lift_baseline (Grasp-Only Lift Reward)

### Motivation

V5's lift_baseline reward allowed the nudge exploit. V6 removes it entirely - only give lift reward when grasping.

### Reward Structure

```python
# V6: Only lift reward when grasping + push-down penalty
reward = 0.0
cube_z = info["cube_z"]

# Reach reward
reward += 1.0 - np.tanh(10.0 * gripper_to_cube)

# Push-down penalty
if cube_z < 0.01:
    reward -= (0.01 - cube_z) * 50.0

# NO lift_baseline - removed in V6

# Grasp bonus + lift reward (ONLY when grasping)
if info["is_grasping"]:
    reward += 0.25
    reward += max(0, (cube_z - 0.01) * 40.0)

# Binary lift bonus
if cube_z > 0.02:
    reward += 1.0

# Success bonus
if info["is_success"]:
    reward += 10.0
```

### Reward Comparison (V5 vs V6)

| cube_z | V5 No Grasp | V5 Grasp | V6 No Grasp | V6 Grasp |
|--------|-------------|----------|-------------|----------|
| 0.007 | 0.60 | 0.85 | 0.75 | 1.00 |
| 0.010 | 0.90 | 1.15 | 0.90 | 1.15 |
| 0.014 | **0.94** | 1.35 | **0.90** | 1.31 |
| 0.020 | 2.00 | 2.65 | 1.90 | 2.55 |
| 0.080 | 2.60 | 5.65 | 1.90 | 5.55 |

Key difference: In V6, nudging cube to z=0.014 without grasping gives **same reward as baseline** (0.90). No incentive to nudge.

### Files

- `runs/lift_cube/20251216_094258/` - V5 training run
- `runs/lift_cube/20251216_094258/eval_v5_500k_combined.mp4` - V5 evaluation video
