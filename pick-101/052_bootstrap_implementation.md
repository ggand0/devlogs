# Bootstrap Implementation for Image-Based RL

## Overview

Following QT-Opt approach: seed replay buffer with scripted policy trajectories to help RL discover successful grasps early.

## Scripts

### `scripts/collect_scripted_grasps.py`

Collects trajectories using a scripted grasp policy.

**Usage:**
```bash
MUJOCO_GL=egl uv run python scripts/collect_scripted_grasps.py \
    --episodes 1000 \
    --output runs/bootstrap/scripted_grasps.pkl
```

**Scripted Policy Phases:**
1. **Descend** (25 steps): Open gripper, move down with noise
2. **Close** (20 steps): Close gripper with small position noise
3. **Lift** (until cube_z >= 0.085): Lift with closed gripper
4. **Hold** (remaining steps): Maintain position until success or max_steps

**Episode Termination:**
- Success: Terminates when `hold_count >= 10` (~60 steps)
- Failure: Runs full 200 steps (truncated)

**Success Rate:** ~80% with curriculum stage 3 (gripper near cube)

**Output Format:**
```python
{
    'trajectories': [
        {
            'trajectory': [
                {'obs': ..., 'action': ..., 'reward': ..., 'terminated': ..., 'truncated': ..., 'info': ...},
                ...
            ],
            'success': bool,
            'episode': int,
            'total_reward': float,
        },
        ...
    ],
    'stats': {
        'num_episodes': int,
        'successes': int,
        'success_rate': float,
        'mean_reward': float,
        'std_reward': float,
        'curriculum_stage': int,
    }
}
```

### `scripts/train_with_bootstrap.py`

Loads scripted trajectories and seeds replay buffer before RL training.

**Usage:**
```bash
# First collect trajectories
MUJOCO_GL=egl uv run python scripts/collect_scripted_grasps.py --episodes 500

# Then train with bootstrap
MUJOCO_GL=egl uv run python scripts/train_with_bootstrap.py \
    --config configs/drqv2_lift_s3_v13.yaml \
    --bootstrap runs/bootstrap/scripted_grasps.pkl
```

**How it works:**
1. Load config and create workspace
2. Load bootstrap trajectories from pickle
3. Add each transition to replay buffer
4. Start normal RL training

## Key Parameters

| Parameter | Value | Notes |
|-----------|-------|-------|
| `target_z` | 0.085m | Just above success threshold (0.08m) |
| `hold_steps` | 10 | Steps to hold for success |
| `max_steps` | 200 | Episode length |
| `noise_scale` | 0.2 | Action perturbation for diversity |
| `curriculum_stage` | 3 | Gripper starts near cube |

## Success Condition

From `lift_cube.py`:
```python
is_grasping = gripper_state < 0.25 and both_finger_contacts
is_lifted = is_grasping and cube_z > lift_height  # 0.08m
is_success = hold_count >= hold_steps  # 10 steps
```

## Why Bootstrap?

From devlog 047 research:
- Random exploration rarely discovers successful grasps
- Sparse rewards provide no gradient toward closing gripper
- QT-Opt achieved 96% success using scripted policy bootstrap with 15-30% initial success rate
- Our scripted policy achieves ~80% success rate

## Visualization

Sample videos saved to `runs/bootstrap/videos_v2/`:
- `ep0_success_61frames.mp4` - typical successful grasp
- Shows descend -> close -> lift -> hold sequence
