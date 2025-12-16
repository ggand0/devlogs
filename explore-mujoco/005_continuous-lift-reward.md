# Continuous Lift Reward Experiment

## Motivation

The v1 Cartesian training (devlog 004) achieved consistent grasping (contacts=(True, True)) but 0% success because:
- Agent approaches from top, contacts cube's top surface + one side
- Binary lift reward (cube_z > 0.02) never triggered since cube at z=0.01, pushed down to 0.007
- No gradient to encourage lifting attempts

Hypothesis: Adding continuous lift reward would incentivize any upward cube movement.

## Changes from v1

1. **Grasp bonus**: 0.25 → 0.5 (stronger incentive to maintain grasp)
2. **Continuous lift reward**: `max(0, (cube_z - 0.01) * 50.0)` instead of binary threshold
3. **Target height bonus**: +2.0 if cube_z > lift_height (0.08m)

```python
# v2 reward structure
reward = 0.0
reward += 1.0 - np.tanh(10.0 * gripper_to_cube)  # Reach
if info["is_grasping"]:
    reward += 0.5  # Grasp (was 0.25)
reward += max(0, (cube_z - 0.01) * 50.0)  # Continuous lift (NEW)
if cube_z > self.lift_height:
    reward += 2.0  # Target height bonus (NEW)
if info["is_success"]:
    reward += 10.0
```

## Training Results

```
Config: configs/lift_cartesian_500k.yaml (same as v1)
Training time: ~1h 54min
Final eval reward: 95.82 +/- 44.76
Success rate: 0%
```

### v1 vs v2 Comparison

| Metric | v1 (robosuite-style) | v2 (continuous lift) |
|--------|---------------------|----------------------|
| Mean reward | 272.28 ± 0.29 | 78.02 ± 53.54 |
| Grasping | Consistent | None |
| Contacts | (True, True) | (False, False) |
| Gripper state | -0.16 (closed) | 1.6 (open) |
| Cube z | 0.007 (pushed down) | 0.010 (unchanged) |
| Behavior | Approach → close → hold | Approach with open gripper, push |

### Behavior Analysis from Video

**Episode patterns observed:**
- **Ep 1-2**: Approaches cube with open gripper, brief closing attempt, reverts to open
- **Ep 3-5**: Pushes cube to the side with bottom of open gripper, then stops moving

The high variance (±53.54) reflects these inconsistent strategies. Some episodes achieve ~140 reward (close approach), others ~30 (stuck far away).

## Why v2 Failed

1. **Continuous lift reward backfired**: Pushing cube can momentarily raise cube_z due to physics, creating brief reward spikes that reinforce erratic pushing behavior

2. **Reward landscape became harder to optimize**: v1's simpler binary structure led to a clear local optimum (approach + grasp + hold). v2's continuous rewards created multiple competing gradients

3. **Open gripper strategy**: Agent learned that approaching with open gripper maximizes reach reward without risking the complexity of grasping. The 0.5 grasp bonus wasn't enough to overcome this local minimum

4. **No contacts means no grasp bonus**: Since agent never closes gripper properly, it never experiences the grasp bonus, so there's no gradient toward grasping behavior

## Key Insight

The continuous lift reward assumed the agent would first learn to grasp (as v1 did), then learn to lift. Instead, it disrupted the grasping learning entirely. The reward changes weren't additive improvements—they changed the optimization landscape.

**Robosuite's staged approach works because:**
- Each stage's reward only activates after previous stage is achieved
- Prevents reward hacking by skipping stages
- Our continuous lift reward violated this by giving reward for any cube_z change

## Next Steps

1. **Staged rewards**: Only give lift reward AFTER grasping is detected
   ```python
   if info["is_grasping"]:
       reward += 0.5
       reward += max(0, (cube_z - 0.01) * 50.0)  # Only when grasping
   ```

2. **Gripper orientation reward**: Encourage approaching from side rather than top

3. **Increase cube friction**: Make gripping more forgiving in physics

4. **Curriculum learning**: Start cube closer to gripper

## Files

- `envs/lift_cube_cartesian.py` - Modified reward function
- `runs/lift_cube_cartesian/20251214_202151/` - v2 training run
- `eval_cartesian_v2_final.mp4` - Evaluation video

---

## Training Infrastructure Updates

### Resume Feature Fix

The original resume implementation had a critical bug: when resuming training, checkpoints were saved to the **same directory** as the original run, causing newer checkpoints to **overwrite** existing ones due to SB3's checkpoint naming (which counts from 0).

**Bug example**: Resuming from 500k checkpoint would save new checkpoints as `50000_steps.zip`, `100000_steps.zip`, etc., overwriting the original early checkpoints.

### New Resume Behavior

When using `--resume`, training now:
1. Creates a **new timestamped directory** with `_resumed` suffix (e.g., `20251215_030000_resumed/`)
2. Loads model weights and VecNormalize stats from the **original** directory
3. Saves all new artifacts (checkpoints, tensorboard, final_model) to the **new** directory
4. Creates `RESUME_INFO.txt` documenting the source checkpoint

**Usage:**
```bash
python train_lift_cartesian.py --resume runs/lift_cube_cartesian/20251214_180426 --timesteps 250000
```

### Other Updates

- **`--timesteps` CLI argument**: Override config timesteps (useful for specifying additional training steps when resuming)
- **Numerical checkpoint sorting**: Checkpoints now sorted by step number, not alphabetically (fixes `50000` being sorted after `500000`)
- **Plot callback**: Auto-saves learning curves at checkpoint intervals
- **Eval env normalization**: Eval environment now loads same VecNormalize stats when resuming

### Checkpoint State Documentation

For `runs/lift_cube_cartesian/20251214_180426/`, see `CHECKPOINT_INFO.txt` for details on which checkpoints were affected by the overwrite bug.

### Current Checkpoint State (20251214_180426)

| Checkpoint | Timestamp | Notes |
|------------|-----------|-------|
| 500000_steps.zip | Dec 14 19:57 | **Original v1** - intact, unaffected |
| 50000-250000_steps.zip | Dec 15 | Overwritten by resumed training |
| 750000_steps.zip | Dec 15 | From resumed training (before fix) |

The original 500k v1 checkpoint is preserved and can be used for future experiments.

### Timestep Counter Fix

Previously, resuming showed `total_timesteps: 800` instead of `500800`. Fixed by:
1. Setting `model.num_timesteps = resume_step` before calling `learn()`
2. Using `reset_num_timesteps=False` to preserve the counter
3. Setting `total_timesteps = resume_step + additional_timesteps` as the target

Now when resuming from 500k with `--timesteps 250000`:
```
Resuming Lift (Cartesian) training from step 500000...
Training for 250000 additional timesteps (target: 750000 total)
```

Logs will show `total_timesteps: 500800, 501600, ...` correctly.
