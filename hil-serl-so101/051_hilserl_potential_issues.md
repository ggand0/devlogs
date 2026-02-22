# HIL-SERL Potential Issues Analysis

## Reward Signal Options

There are TWO ways to provide reward signals in HIL-SERL:

### Option 1: Manual Success Marking (No classifier needed)

Press `k` during training to mark episode as success:
- `GearedLeaderControlWrapper` sets `reward=1.0` and terminates episode
- Q-learning backpropagates this reward through the episode via `discount=0.99`

**No reward classifier required.** Just press `k` when the cube is lifted.

### Option 2: Automatic Reward Classifier

Set `reward_classifier_pretrained_path` to automatically detect success every step.

---

## FIXED: Robot Drifting Right With No Learning

### Problem
Robot only drifted right after the first ~100 steps of random exploration.

### Root Cause: Coordinate Frame Mismatch

**Genesis sim** (scene_config.py):
```python
ROBOT_POS = (0.25, 0.314, 0.0)
ROBOT_EULER = (0, 0, -90)  # Robot faces -Y direction
```

**Real robot MuJoCo** (so101_new_calib.xml):
```xml
<body name="base" pos="0 0 0" quat="1 0 0 0">  <!-- Robot at origin, faces +X -->
```

The robots face different directions:
- **Sim**: Robot faces **-Y** (rotated -90° around Z)
- **Real**: Robot faces **+X** (no rotation)

When sim policy outputs `delta_y < 0` (move forward in sim toward -Y), the real robot moves in **-Y world direction = RIGHT**.

### Fix
Added `Sim2RealActionTransformWrapper` that rotates actions by +90° in XY plane:
```python
real_action[0] = -sim_action[1]  # real X = -sim Y
real_action[1] = sim_action[0]   # real Y = sim X
```

Enable in config:
```json
"sim2real_action_transform": true
```

### Files Modified
- `lerobot/scripts/rl/gym_manipulator.py`: Added `Sim2RealActionTransformWrapper`
- `lerobot/envs/configs.py`: Added `sim2real_action_transform` option
- `train_config.json`: Enabled the transform

### Note
This same transformation already exists in `scripts/rl_inference.py` (lines 712-721) under the `--genesis_mode` flag. The wrapper was added to `gym_manipulator.py` because HIL-SERL uses a different code path.

### Offline Dataset Transformation

The offline dataset was recorded in **real-frame**, but the policy operates in **sim-frame**. To maintain consistency:

1. Created `scripts/transform_dataset_actions.py` to apply the INVERSE transform to offline data:
   ```python
   sim_action[0] = real_action[1]   # sim X = real Y
   sim_action[1] = -real_action[0]  # sim Y = -real X
   ```

2. Transformed 1574 frames across 18 episodes

3. Updated train_config.json to use transformed dataset:
   ```json
   "repo_id": "gtgando/so101_pick_lift_cube_locked_wrist_sim_frame"
   ```

This ensures offline data and online policy actions are in the same coordinate frame.

---

## FIXED: Intervention Actions Frame Mismatch

### Problem
During HIL-SERL, intervention actions from the leader arm were recorded in REAL-FRAME while policy outputs are in SIM-FRAME. This causes the Q-function to learn inconsistent values.

### Flow Before Fix
1. Policy outputs `action` (SIM-FRAME)
2. `Sim2RealActionTransformWrapper.action()` transforms to REAL-FRAME for robot
3. Intervention detected → `info["action_intervention"]` is in REAL-FRAME
4. Actor records `info["action_intervention"]` directly (REAL-FRAME)
5. **Mismatch**: Policy samples from buffer expecting SIM-FRAME actions

### Fix
Added `step()` method to `Sim2RealActionTransformWrapper` that inverse-transforms intervention actions:
```python
def step(self, action):
    obs, reward, done, truncated, info = super().step(action)

    if "action_intervention" in info and info["action_intervention"] is not None:
        intervention = info["action_intervention"]
        # Inverse: sim_x = real_y, sim_y = -real_x
        transformed[0] = intervention[1]
        transformed[1] = -intervention[0]
        info["action_intervention"] = transformed

    return obs, reward, done, truncated, info
```

Now all actions (policy + intervention) are recorded in SIM-FRAME.

---

## Verified: These Are NOT Issues

### Action Shape Handling
`TorchActionWrapper` correctly squeezes (1, 4) → (4,) before env.step(). No fix needed.

### Observation Key Mapping
`preprocess_observation()` in lerobot/envs/utils.py correctly maps:
- `agent_pos` → `observation.state`

The `FullProprioceptionWrapper` populates `agent_pos` with 18-dim state, which is then renamed by `preprocess_observation()`.

### Frame Stacking
`FrameStackBuffer` uses `state_key="observation.state"` which matches the output of `ConvertToLeRobotObservation`.

---

## Summary

| Issue | Severity | Status |
|-------|----------|--------|
| No reward classifier | Non-issue | Manual `k` key works |
| Robot drifting right | CRITICAL | FIXED - coordinate frame transform |
| Offline/online frame mismatch | HIGH | FIXED - dataset transformed to sim-frame |
| Intervention action frame mismatch | CRITICAL | FIXED - inverse transform in step() |
| Action shape | Non-issue | Verified working |
| Observation keys | Non-issue | Verified working |
| Frame stacking | Non-issue | Verified working |

## Controls During Training

```
k - Episode SUCCESS (reward=1.0, terminates)
e - Episode END (reward=0.0, terminates)
i - Toggle intervention mode
```
