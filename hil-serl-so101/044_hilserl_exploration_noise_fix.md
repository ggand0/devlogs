# Devlog 044: HIL-SERL Training Fixes

**Date**: 2025-01-18
**Status**: Complete

## Problem

During HIL-SERL training with DrQ-v2, the robot arm exhibited constant shaking with no purposeful movement toward the cube. The pretrained Genesis policy behavior was not visible despite loading weights correctly.

Multiple issues were discovered:
1. Excessive exploration noise masking pretrained policy
2. Checkpoint saving failed with frame stacking enabled
3. Offline dataset had zero actions (no learning signal)

## Root Cause Analysis

### Issue 1: Excessive Exploration Noise

The `stddev_schedule` was set to `linear(1.0,0.1,50000)`:
- Initial stddev = 1.0 (extremely high)
- Final stddev = 0.1
- Decay over 50000 steps

At step 1000, the stddev was still ~0.98, meaning:
- Actions were essentially **random noise** added to policy outputs
- The pretrained Genesis behavior was completely **masked** by exploration
- Robot appeared to shake randomly instead of executing learned movements

### Issue 2: Frame Stacking vs LeRobotDataset Incompatibility

When saving checkpoints, the learner tried to convert the replay buffer to LeRobotDataset format. With 3-frame stacking:
- Images had 9 channels (3 RGB × 3 frames)
- LeRobotDataset stats validation expects 3 channels
- Result: `ValueError: Shape of 'min' must be (3,1,1), but is (9, 84, 84)`

### Issue 3: Zero Actions in Offline Dataset (Critical)

The offline dataset (`so101_pick_lift_cube_merged`) had **all zero actions**:

```
Action stats: min=[0. 0. 0. 0.], max=[0. 0. 0. 0.]
```

This was caused by how recording worked when `kinematics=None`:

```python
# In BaseLeaderControlWrapper._handle_intervention()
if self.kinematics is not None:
    # Compute EE delta actions via FK
    action = np.clip(leader_ee - follower_ee, ...)
else:
    # Fallback: joint mirroring, but action is DUMMY ZEROS
    self.unwrapped._leader_positions = {...}  # Joint mirroring works
    action = np.array([0.0, 0.0, 0.0])  # Action stored is zeros!
```

During recording:
- Robot moved correctly via direct joint mirroring (`_leader_positions`)
- But the `action` field stored was dummy zeros
- The offline buffer had no learning signal from demonstrations

The Q-learning critic loss was extremely high (11k-28k) because:
- Policy was trying to fit pretrained Q-values
- But all actions in offline buffer were zeros
- Massive value function mismatch

## Fixes Applied

### Fix 1: Reduce Exploration Noise

Changed `train_config.json`:

```json
// Before
"stddev_schedule": "linear(1.0,0.1,50000)"

// After
"stddev_schedule": "linear(0.2,0.1,10000)"
```

Now:
- Initial stddev = 0.2 (controlled exploration)
- Final stddev = 0.1
- Faster decay over 10000 steps

This allows the pretrained policy to express its learned behavior while still exploring for fine-tuning.

### Fix 2: Graceful Checkpoint Error Handling

Modified `learner.py` to catch the frame stacking shape error:

```python
try:
    replay_buffer.to_lerobot_dataset(repo_id=repo_id_buffer_save, fps=fps, root=dataset_dir)
except ValueError as e:
    if "Shape of" in str(e):
        logging.warning(f"[LEARNER] Skipping buffer-to-dataset conversion (frame stacking incompatible): {e}")
    else:
        raise
```

Policy weights are still saved correctly; only the buffer-to-dataset conversion is skipped.

### Fix 3: Compute Actions from State Changes via FK

Modified `learner.py` to detect zero actions and trigger FK-based action computation:

```python
# Check if dataset actions are all zeros (corrupted/missing actions)
actions_are_zero = False
if len(offline_dataset) > 0:
    sample_actions = torch.stack([offline_dataset[i]["action"] for i in range(min(100, len(offline_dataset)))])
    if sample_actions.abs().max() < 1e-6:
        actions_are_zero = True
        logging.warning("[LEARNER] Dataset actions are all zeros - will compute from state changes")

# Trigger FK-based action computation
if actions_are_zero and mujoco_model_path is not None:
    convert_actions_to_ee = True
    logging.info(f"Computing actions from state changes via FK (scale: {ee_action_scale})")
```

Modified `buffer.py` to compute gripper action from joint positions instead of zero action:

```python
# Gripper action: compute from gripper joint position change
if len(prev_joints) >= 6:
    prev_gripper = prev_joints[5]  # Gripper joint position in degrees
    curr_gripper = curr_joints[5]
    gripper_delta = curr_gripper - prev_gripper
    gripper_action = np.clip(gripper_delta / 50.0, -1.0, 1.0)
```

Now the offline buffer is populated with proper EE delta actions computed from consecutive joint positions via MuJoCo forward kinematics.

## Why HIL-SERL Needs Lower Initial Noise

| Training Scenario | Recommended Initial Stddev |
|-------------------|---------------------------|
| RL from scratch | 1.0 (need random exploration) |
| HIL-SERL with demos | 0.3-0.5 (policy has structure) |
| HIL-SERL with pretrained | 0.1-0.2 (policy already works) |

With a pretrained Genesis policy that can already pick cubes in simulation, starting with stddev=1.0 destroys all learned behavior and wastes training time re-learning basic skills.

## Files Modified

- `outputs/hilserl_drqv2/train_config.json` - Reduced stddev_schedule
- `lerobot/src/lerobot/scripts/rl/learner.py` - Added try-except for checkpoint buffer conversion, zero action detection
- `lerobot/src/lerobot/utils/buffer.py` - Fixed gripper action computation for FK-based action conversion

## Key Insights

### Exploration Noise

HIL-SERL's purpose is to **fine-tune** a policy with human interventions, not to train from scratch. The exploration noise should be calibrated to:
1. Allow the pretrained policy to demonstrate its learned behavior
2. Provide enough exploration to improve from demonstrations
3. Not so much that it masks the policy entirely

### Dataset Recording

When recording teleoperation datasets for HIL-SERL:
1. **Ensure kinematics is properly configured** during recording
2. If using `so101_follower_end_effector`, the MuJoCo model path must be set
3. Without kinematics, actions are stored as zeros (joint mirroring still works but actions are lost)
4. The offline buffer can recover actions post-hoc via FK if joint positions are stored

### Debugging High Critic Loss

If critic loss is extremely high (>10000):
- Check if offline buffer actions are zeros: `sample_actions.abs().max() < 1e-6`
- Verify the action conversion is triggering: look for "Computing actions from state changes via FK" log
- Delete cached offline buffer (`offline_buffer.pt`) to force rebuild

## Related

- Devlog 042: DrQ-v2 Encoder Synchronization Fix
- Devlog 043: Teleoperation Unit Mismatch Fix
