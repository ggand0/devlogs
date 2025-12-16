# Devlog: Kernel Upgrade and Training Resume

**Date**: 2024-12-13

## Issue

Training crashed mid-run on the AMD RX 7900 XTX GPU. This is a known stability issue with the 7900 XTX on older kernels.

## Fix: Kernel Upgrade

Upgraded Ubuntu kernel from `6.2.0` to `6.8.0-90-generic` via SSH while the system was unresponsive locally.

The 6.8 kernel includes fixes for RDNA3 GPUs (7900 XTX).

## Verification

After reboot, verified PyTorch ROCm still works:

```bash
uv run python -c "import torch; print(f'PyTorch: {torch.__version__}'); print(f'ROCm available: {torch.cuda.is_available()}'); print(f'Device name: {torch.cuda.get_device_name(0)}')"
```

Output:
```
PyTorch: 2.9.1+rocm6.4
ROCm available: True
Device name: Radeon RX 7900 XTX
```

GPU tensor operations test:
```bash
uv run python -c "
import torch
x = torch.randn(1000, 1000, device='cuda')
y = torch.randn(1000, 1000, device='cuda')
z = torch.mm(x, y)
print(f'Matrix multiply: {z.shape}, sum: {z.sum().item():.4f}')
"
```

Result: GPU compute works.

## Resuming Training

Training crashed at the 120k checkpoint. To resume:

```bash
uv run train.py --timesteps 500000 --resume runs/checkpoints/sac_pick_cube_120000_steps.zip
```

### How to verify resume worked

The log shows `total_timesteps` starting from 0, which looks like a fresh start. However, check the `n_updates` field:

```
| train/             |          |
|    n_updates       | 119598   |
```

This confirms the model's internal state was restored from the checkpoint (~120k updates). The timestep counter just tracks the current run's progress toward the goal.

### Finding the latest checkpoint

```bash
ls -lt runs/checkpoints/ | head -5
```

The most recent file is where training stopped.

## Training Results (500k+ total steps)

After resuming and completing training (~500k total steps: 120k before crash + 380k after resume):

### Final Training Metrics
```
| rollout/           |          |
|    success_rate    | 0        |
| time/              |          |
|    episodes        | 1900     |
|    fps             | 67       |
|    time_elapsed    | 5607     |
|    total_timesteps | 380000   |
| train/             |          |
|    actor_loss      | 1.62     |
|    critic_loss     | 0.000218 |
|    ent_coef        | 0.000137 |
|    n_updates       | 497998   |

Eval num_timesteps=380000, episode_reward=-53.09 +/- 7.04
Episode length: 200.00 +/- 0.00
Success rate: 0.00%
```

Training time: ~1.5 hours for 380k steps (post-resume)

### Evaluation Results
```
$ uv run eval.py --record runs/videos --episodes 5

Episode 1: reward=-103.66, steps=200, success=False
Episode 2: reward=-99.14, steps=200, success=False
Episode 3: reward=-54.18, steps=200, success=False
Episode 4: reward=-48.48, steps=200, success=False
Episode 5: reward=-50.93, steps=200, success=False

Results over 5 episodes:
  Success rate: 0/5 (0.0%)
  Mean reward: -71.28
```

### Analysis

The agent learned to move toward the cube but failed to grasp it. With 0% success rate across all training, this confirms the challenge of pick-and-place with simple distance-based rewards:

1. **Dense rewards encourage pushing, not grasping** - The agent gets reward for moving the cube closer to the target, so it pushes rather than picks
2. **No incentive to close gripper** - The current reward doesn't distinguish between gripper open/closed
3. **Sparse success signal** - Only getting reward when cube is at target is too rare during random exploration

## Improvements Implemented

Based on research (see `docs/pick-place-rl-research.md`), implemented two approaches:

### 1. Staged Rewards with Grasp Detection

Updated `envs/pick_cube.py` with 5-stage reward system:

1. **Reach** (-distance): Reward for gripper approaching cube
2. **Grasp** (+1.0): Bonus when gripper closes around cube
3. **Lift** (+2.0 + height×10): Bonus when cube is lifted while grasping
4. **Place** (-2×distance): Only rewards moving toward target when lifted
5. **Success** (+10.0): Bonus when cube reaches target

Key insight: Place reward only applies when lifted, so the agent can't cheat by pushing.

Config: `configs/staged_rewards_1m.yaml`
```bash
uv run train.py --config configs/staged_rewards_1m.yaml
```

### 2. HER (Hindsight Experience Replay)

Created `envs/pick_cube_goal.py` - a GoalEnv version compatible with HER:

- Dict observation space: `observation`, `achieved_goal`, `desired_goal`
- Vectorized `compute_reward()` for goal relabeling
- Sparse (-1/0) rewards work well with HER

**How HER works:**
1. Agent tries to place cube at target position `(0.4, 0.1, 0.04)`
2. Agent fails - cube ends up at `(0.35, 0.05, 0.01)`
3. HER creates virtual episode: "what if goal was `(0.35, 0.05, 0.01)`?" → Success!
4. Agent learns from virtual success

This gives learning signal even when the agent never reaches the actual goal.

Config: `configs/her_1m.yaml`
```bash
uv run train_her.py --config configs/her_1m.yaml
```

**HER Results (500k steps):** Failed. Flat -200 reward (max episode length × -1) across all evaluations. 0% success rate.

The arm pointed toward mid-air with gripper open, never approaching or touching the cube. HER couldn't help because:
- The agent never contacts the cube through random exploration
- Every "virtual success" is just "goal = cube's initial position" (unchanged)
- No gradient toward reaching/grasping behavior

**Why HER failed here but works elsewhere:**

HER works when random exploration naturally produces varied outcomes:
- **Pushing tasks**: Random movements accidentally bump objects to different positions
- **Fetch (OpenAI Gym)**: Gripper starts close to object, noise = accidental contact
- **Navigation**: Robot ends up in different (x,y) just by moving

Our setup: cube is 40cm away, 6-DOF joint space random actions = arm waving in place. Probability of accidentally touching a 2cm cube ≈ 0. No contact = cube never moves = no goal diversity = HER generates fake "successes" for the same initial cube position every time.

HER papers often use dense reach rewards alongside HER, or start the gripper near the object. Pure sparse rewards + distant object = exploration problem HER can't solve.

## Notes

- Checkpoints saved every 10k steps
- Resume loads: model weights, optimizer state, replay buffer, normalization stats
- The 500k timesteps flag means 500k *additional* steps from the checkpoint
