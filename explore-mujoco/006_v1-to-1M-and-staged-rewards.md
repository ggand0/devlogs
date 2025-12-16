# V1 Training to 1M and Staged Rewards Plan

## V1 Training Summary

### Original V1 (500k steps)
```
Directory: runs/lift_cube_cartesian/20251214_180426/
Checkpoint: sac_lift_cartesian_500000_steps.zip
Training time: ~2 hours
```

**Architecture:**
- Action space: Cartesian (delta XYZ + gripper) - 4 dims
- Observation space: 21 dims (joint pos/vel + gripper pos/euler + cube pos)
- IK controller internally converts to joint controls

**Reward function (robosuite-style):**
```python
reward = 0.0
reward += 1.0 - np.tanh(10.0 * gripper_to_cube)  # Reach
if is_grasping:
    reward += 0.25  # Grasp bonus
if cube_z > 0.02:
    reward += 1.0  # Binary lift bonus
if is_success:
    reward += 10.0
```

**Results at 500k:**
- Mean reward: ~272
- Consistent grasping behavior (contacts True/True)
- 0% success rate (cube pushed down, not lifted)

### Resumed Training to 1M

```
Directory: runs/lift_cube_cartesian/20251215_090951_resumed/
Resumed from: 500k v1 checkpoint
Additional steps: 500k (total: 1M)
Training time: ~2 hours
```

**Results at 1M:**
- Mean reward: ~225
- Still consistent grasping
- Still 0% success rate

### Evaluation Analysis (eval_1M.mp4)

**Observed behavior across 5 episodes:**

| Episode | Gripper State | Contacts | Cube Z | Behavior |
|---------|--------------|----------|--------|----------|
| 1 | -0.16 (closed) | (True, True) | 0.007 | Grasp from top, push down |
| 2 | -0.16 (closed) | (True, True) | 0.007 | Same pattern |
| 3 | -0.16 (closed) | (True, True) | 0.007 | Same pattern |
| 4 | -0.16 (closed) | (True, True) | 0.007→0.015 | Cube popped up when released |
| 5 | -0.16 (closed) | (True, True) | 0.007 | Same pattern |

**Key observations:**
1. Agent consistently approaches cube and closes gripper
2. Contacts both static gripper and moving jaw (is_grasping=True)
3. Cube pushed DOWN to z=0.007 instead of lifted (baseline z=0.010)
4. In Episode 4, cube popped up to z=0.015 when grasp released - but this didn't trigger reward (threshold is z > 0.02)

**Why it's stuck:**
- Approaching from top → can only push cube down
- Binary lift reward (z > 0.02) provides no gradient from z=0.007
- Agent found local optimum: reach + grasp rewards without lift

## Staged Rewards Plan (V3)

### Motivation

The v1 agent learned to grasp but not lift because:
1. No reward signal for small upward movements
2. Binary threshold (z > 0.02) unreachable from current approach angle

### New Reward Structure

```python
# envs/lift_cube.py
def _compute_reward(self, info):
    reward = 0.0
    cube_z = info["cube_z"]

    # Reach: encourage gripper to approach cube
    gripper_to_cube = info["gripper_to_cube"]
    reach_reward = 1.0 - np.tanh(10.0 * gripper_to_cube)
    reward += reach_reward

    # Small reward for any cube height above baseline (exploration signal)
    # Even without grasp, launching the cube gives a small signal
    lift_baseline = max(0, (cube_z - 0.01) * 10.0)
    reward += lift_baseline

    # Grasp bonus + stronger lift reward when grasping
    if info["is_grasping"]:
        reward += 0.25  # Grasp bonus
        # Additional lift reward when grasping (4x stronger than baseline)
        reward += max(0, (cube_z - 0.01) * 40.0)

    # Binary lift bonus (same as v1)
    if cube_z > 0.02:
        reward += 1.0

    # Success bonus
    if info["is_success"]:
        reward += 10.0

    return reward
```

### Reward at Different Heights

| cube_z | Without Grasp | With Grasp | Notes |
|--------|--------------|------------|-------|
| 0.010 | 0.0 | 0.25 | Baseline (cube resting) |
| 0.015 | 0.05 | 0.45 | Episode 4 pop-up level |
| 0.020 | 0.10 + 1.0 | 0.65 + 1.0 | Binary threshold |
| 0.050 | 0.40 + 1.0 | 1.85 + 1.0 | Mid-lift |
| 0.080 | 0.70 + 1.0 | 3.05 + 1.0 | Success height |

### Why This Should Work

1. **Exploration signal**: Small reward for ANY cube elevation encourages trying different approaches
2. **Grasp multiplier**: 4x stronger lift reward when grasping incentivizes lifting while holding
3. **Gradient from current state**: z=0.015 now gives +0.45 reward (vs 0 in v1)
4. **Preserves v1 learning**: Still rewards reaching and grasping

### Training Plan

Resume from 1M v1 checkpoint with new reward:

```bash
uv run python train_lift.py \
    --resume runs/lift_cube_cartesian/20251215_090951_resumed \
    --timesteps 500000
```

This will:
- Load 1M checkpoint (already knows how to reach and grasp)
- Use new staged reward from `envs/lift_cube.py`
- Train for 500k additional steps
- Save to `runs/lift_cube/{timestamp}_resumed/`

### Expected Outcome

The agent should:
1. Initially see reward DROP (new reward structure different from v1)
2. Quickly re-learn reaching and grasping (preserved in weights)
3. Discover lifting gives more reward than pushing down
4. Learn to approach from angle that allows upward movement

### Files

- `envs/lift_cube.py` - New environment with staged rewards
- `train_lift.py` - Training script (updated with resume fixes)
- `configs/lift_500k.yaml` - Config for training

### Checkpoints Reference

**V1 Original (500k):**
```
runs/lift_cube_cartesian/20251214_180426/checkpoints/sac_lift_cartesian_500000_steps.zip
```

**V1 Extended (1M):**
```
runs/lift_cube_cartesian/20251215_090951_resumed/checkpoints/sac_lift_cartesian_1000000_steps.zip
```

---

## V3 Staged Rewards Results (FAILED)

### Training Run
```
Directory: runs/lift_cube/20251215_134243_resumed/
Resumed from: V1 1M checkpoint
Additional steps: 500k (total: 1.5M)
Training time: ~2 hours
```

### Mid-Training Eval (1.25M steps)

| Episode | Gripper | Contacts | Cube Z | Behavior |
|---------|---------|----------|--------|----------|
| 1 | -0.166 (closed) | (True, True) | 0.007 | Old v1 behavior - push down |
| 2 | +0.983 (open) | (False, False) | 0.010 | NEW: hover with open gripper |
| 3 | -0.162 (closed) | (True, True) | 0.007 | Old v1 behavior |
| 4 | +0.251 (open) | (False, False) | 0.010 | Hover behavior |
| 5 | -0.161 (closed) | (True, True) | 0.007 | Old v1 behavior |

- Mean reward: 190.93 (dropped from ~225)
- Success rate: 0%
- Policy destabilizing, alternating between two strategies

### Final Eval (1.5M steps)

| Episode | Gripper | Contacts | Cube Z | Behavior |
|---------|---------|----------|--------|----------|
| 1 | +0.375 (open) | (False, False) | 0.010 | Hover near cube |
| 2 | Mixed | Mixed | 0.010 | Grasp then release |
| 3 | Mixed | Mixed | 0.008 | Unstable |
| 4 | Mixed | Mixed | 0.013 | Brief lift attempt? |
| 5 | +0.650 (open) | (False, False) | 0.010 | Hover |

- Mean reward: 146.56 +/- 21.28 (worse than v1's ~225)
- Success rate: 0%
- High variance indicates unstable policy

### Observed Exploration Behaviors

Video analysis shows the agent actively exploring different manipulation strategies:

1. **Fiddling with cube** - Agent repeatedly contacts and releases the cube, appearing to test different interaction modes
2. **Launch attempts** - Some episodes show the agent making quick contact motions that could flick/launch the cube upward
3. **Grasp-release cycles** - Agent grasps, holds briefly, then releases - possibly exploring what happens when cube is released
4. **Approach angle variation** - Not always top-down; some episodes show varied approach trajectories

This exploration is **positive signal** - the agent is searching the action space rather than fully converging to a single strategy. The high variance (±21) reflects this active exploration rather than pure instability.

### Why V3 Resumed Training Failed

1. **Reward normalization mismatch**: VecNormalize was trained on v1 reward distribution. V3's different reward scale caused instability.

2. **Gradient too weak**: The lift gradient (10.0 and 40.0 multipliers) wasn't strong enough to overcome the local optimum.

3. **V1 priors too strong**: Resuming from v1 checkpoint meant the agent had strong priors for top-down approach. 500k steps wasn't enough to unlearn these.

4. **Hover exploit discovered**: Agent found that hovering with open gripper gives ~0.9 reach reward without risking push-down penalty.

### Key Insight

The agent discovered that **not grasping** avoids the negative signal from pushing the cube down. This is a valid exploitation of the reward function - hover near cube = high reach reward, no grasp penalty for pushing cube down.

However, the exploration behaviors suggest the agent **is trying** to find better strategies - it just hasn't discovered that lifting while grasping is the optimal path. Fresh training (without v1 biases) may allow the agent to discover lifting earlier in training.

---

## V3 Fresh Training Results (FAILED)

### Training Run
```
Directory: runs/lift_cube/20251215_160643/
Fresh training (no resume)
Timesteps: 500k
Training time: ~2 hours
```

### Results at 500k
- Mean reward: 212.76 ± 14.62
- Success rate: 0%
- Same behavior as v1: grasp and push down to z=0.007

### Why V3 Fresh Also Failed

Same local optimum as v1 - the standalone grasp bonus (+0.25) makes holding without lifting too attractive. The agent gets ~1.15/step for grasping (reach + grasp bonus) vs attempting to lift.

---

## V4: Remove Standalone Grasp Bonus (FAILED)

### Motivation

V3's grasp bonus rewarded grasping even when pushing cube down. V4 removes standalone grasp bonus - only reward grasping when cube is elevated.

### Reward Structure

```python
# V4: No standalone grasp bonus - only reward grasping when lifting
if info["is_grasping"] and cube_z > 0.01:
    reward += 0.25  # Grasp bonus only when elevated
    reward += (cube_z - 0.01) * 40.0
```

### Reward Comparison

| cube_z | V3 (Grasp) | V4 (Grasp) | V4 (No Grasp) |
|--------|------------|------------|---------------|
| 0.007 | ~1.15 | ~0.9 | ~0.9 |
| 0.010 | ~1.15 | ~0.9 | ~0.9 |
| 0.015 | ~1.35 | ~1.35 | ~0.95 |
| 0.020 | ~2.65 | ~2.65 | ~2.0 |

Key change: Grasping at z=0.007 now gives same reward as hovering (~0.9).

### Training Run
```
Directory: runs/lift_cube/20251215_190304/
Fresh training
Timesteps: 500k
Training time: ~1.8 hours
```

### Results at 500k

| Episode | Gripper | Contacts | Cube Z | Behavior |
|---------|---------|----------|--------|----------|
| 1 | +0.92 (open) | (True, False) | 0.007 | Hover open, push down |
| 2 | +1.00 (open) | (True, False) | 0.007 | Hover open, push down |
| 3 | +0.87 (open) | (True, False) | 0.007 | Hover open, push down |
| 4 | +0.76 (open) | (True, False) | 0.007 | Hover open, push down |
| 5 | +1.22 (open) | (True, False) | 0.007 | Hover open, push down |

- Mean reward: 176.39 ± 4.26
- Success rate: 0%
- Agent **never closes gripper** - found new exploit

### Why V4 Failed

**New exploit discovered**: By keeping gripper open, agent gets reach reward (~0.9) without any risk. Since standalone grasp bonus was removed, there's no incentive to close gripper at all.

The agent contacts only the static gripper part `(True, False)` - never pinching with the jaw. This is "safer" than grasping because:
1. No risk of grasp failing
2. Same reward as hovering near cube
3. Lower variance in outcomes

### Key Insight

The fundamental problem remains: **pushing down gives same reward as baseline** because both z=0.007 and z=0.010 are below the lift threshold. Without a penalty for pushing down, the agent has no incentive to avoid it.

---

## Next Steps

Options to try:

1. **V5: Negative reward for pushing down** - Penalize cube_z < 0.01 to make push-down strategies unprofitable
2. **Curriculum: elevated cube** - Start with cube at z=0.03-0.05 sometimes so agent experiences lift reward
3. **Different approach**: HER (Hindsight Experience Replay) for goal-conditioned learning
4. **Approach angle constraint** - Force side approach in early training
