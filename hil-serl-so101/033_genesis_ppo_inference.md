# Devlog 033: Genesis PPO Inference Implementation

**Date**: 2025-01-11
**Status**: Complete

## Summary

Implemented inference script for Genesis-trained PPO policy on the real SO-101 robot. The Genesis PPO model differs from DrQ-v2 in that it uses single frames (no frame stacking).

## Files Created/Modified

### New Files
- `scripts/ppo_inference.py` - Main inference script for Genesis PPO

### Modified Files
- `src/deploy/policy.py` - Added GenesisPPORunner, CNNEncoder, ActorCritic classes

## Key Differences: Genesis PPO vs DrQ-v2

| Aspect | DrQ-v2 | Genesis PPO |
|--------|--------|-------------|
| Frame stacking | 3 frames | None (single frame) |
| RGB input shape | `(3, 3, 84, 84)` | `(3, 84, 84)` |
| Low-dim input shape | `(3, 18)` | `(18,)` |
| Architecture | DrQ-v2 encoder | Nature CNN |
| Checkpoint format | Hydra-based | Simple state_dict |
| Reset position | Above cube (Z=0.05) | At grasp height (Z=0.02) |

## Model Architecture

```
Image (3, 84, 84) uint8
        |
        v
CNNEncoder (Nature CNN)
  - Conv2d(3, 32, 8, stride=4) + ReLU
  - Conv2d(32, 64, 4, stride=2) + ReLU
  - Conv2d(64, 64, 3, stride=1) + ReLU
  - Flatten -> Linear(3136, 256) + ReLU
        |
        v
    256 features
        |
        +--------------------+
        |                    |
        v                    v
Low-dim (18,)          Actor + Critic
        |
        v
Low-dim Encoder
  - Linear(18, 64) + ReLU
  - Linear(64, 64) + ReLU
        |
        v
    64 features
        |
        +-----> Concat (320 features)
                      |
          +-----------+-----------+
          |                       |
          v                       v
    Actor Mean              Critic
    Linear(320, 256)        Linear(320, 256)
    Linear(256, 4)          Linear(256, 1)
          |
          v
    Action (4,) in [-1, 1]
```

## Reset Position Fix

Genesis training `curriculum_stage=3` ends at **grasp height**, not above cube:

```python
# Genesis _reset_gripper_near_cube() sequence:
# Phase 2: Move above cube (Z = cube_z + 0.005 + 0.03 = 0.05)
# Phase 3: Move down to grasp height (Z = cube_z + 0.005 = 0.02)  <-- Training starts HERE

# Correct reset for PPO inference:
initial_target = [0.25, -0.015, 0.02]  # At grasp height
```

## Safety Features

1. **Action clipping**: `np.clip(action, -1.0, 1.0)`
2. **Configurable action scale**: `--action_scale 0.01` for safer testing
3. **Ctrl-C handling**: safe_return() lifts arm before shutdown
4. **Joint offset correction**: `ELBOW_FLEX_OFFSET_RAD = -12.5°`

## Usage

```bash
# Conservative first run
uv run python scripts/ppo_inference.py \
    --checkpoint /path/to/checkpoint.pt \
    --action_scale 0.01 \
    --episode_length 50 \
    --num_episodes 1 \
    --control_hz 5.0

# Full speed after verification
uv run python scripts/ppo_inference.py \
    --checkpoint /path/to/checkpoint.pt \
    --action_scale 0.02
```

## Checkpoint Location

```
/home/gota/ggando/ml/pick-101-genesis/runs/ppo_multi_env_wrist/20260111_203814/checkpoints/checkpoint_100352.pt
```

## Dependencies

The implementation is **self-contained** in:
- `scripts/ppo_inference.py` - inference script with all reset logic
- `src/deploy/policy.py` - GenesisPPORunner class added at end of file

Uses existing infrastructure:
- `src/deploy/camera.py` - CameraPreprocessor (unchanged)
- `src/deploy/robot.py` - SO101Robot/MockSO101Robot (unchanged)
- `src/deploy/controllers/ik_controller.py` - IKController (unchanged)

## Branch Note

This was developed on `feat/drqv2-lerobot` branch. The ppo_inference.py script copies the joint offset correction from rl_inference.py so it's self-contained. The GenesisPPORunner addition to policy.py should merge cleanly with main as it's appended at the end of the file.
