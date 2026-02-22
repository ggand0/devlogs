# Devlog 058: HIL-SERL Scratch Training Analysis

Investigation of DrQ-v2 policy training from scratch without sim-pretrained checkpoint. Documents Q-value overestimation, sparse reward challenges, and training dynamics.

## Context

**Goal**: Train DrQ-v2 policy from scratch using HIL-SERL (Human-in-the-Loop RL) without Genesis sim-pretrained checkpoint.

**Motivation**: Genesis-trained policy exhibited bias toward center of workspace due to randomized cube spawn positions during sim training. Training from scratch aims to eliminate this bias.

**Config**: `/home/gota/ggando/ml/so101-playground/outputs/hilserl_drqv2_scratch/train_config.json`

## Key Configuration

```json
{
    "dataset": {
        "repo_id": "gtgando/so101_pick_lift_cube_locked_wrist"  // REAL-frame dataset
    },
    "env": {
        "wrapper": {
            "sim2real_action_transform": false  // No coordinate transform (REAL-frame)
        }
    },
    "policy": {
        "type": "drqv2",
        "pretrained_path": null,  // NO pretrained checkpoint
        "discount": 0.99,
        "stddev_schedule": "linear(0.2,0.05,5000)",
        "utd_ratio": 1,
        "num_explore_steps": 500,
        "frame_stack": 3
    }
}
```

## Frame Consistency

**Critical**: All data must be in the same coordinate frame.

| Component | Frame | Notes |
|-----------|-------|-------|
| Offline dataset | REAL | `gtgando/so101_pick_lift_cube_locked_wrist` |
| Online collection | REAL | `sim2real_action_transform: false` |
| Intervention actions | REAL | Recorded directly from teleoperation |

**Previous Bug**: Used SIM-frame dataset with `sim2real_action_transform: false`, causing frame mismatch between offline and online data.

## Training Statistics (Step 8500)

```
Online buffer size: 5073 transitions
Offline buffer size: 793 transitions
Total transitions: 5866

Positive rewards (offline): 18 (2.3% of offline)
Positive rewards (online): 35 (0.7% of online)
Total positive rewards: 53 (0.9% of total)

Intervention ratio: ~37% of online steps
Successful episodes: ~35-40
```

## Q-Value Overestimation Problem

**Observation**: Actor loss consistently around -1.08

```
Actor loss = -min_q.mean() = -1.08
```

This implies average Q-values > 1.0, which is **theoretically impossible**:

**Theoretical Analysis**:
- Terminal reward: 1.0 (only at episode success)
- Discount factor: 0.99
- Episode length: ~100-150 steps
- Maximum Q-value at any state = 0.99^(steps_to_success)
- For 100 steps: max Q = 0.99^100 ≈ 0.37
- For 50 steps: max Q = 0.99^50 ≈ 0.61

**Root Cause**: Q-value overestimation due to:
1. Sparse terminal rewards - critic rarely sees reward signal
2. Bootstrapping from noisy Q-estimates propagates errors
3. Actor exploits overestimated Q-regions
4. No conservative penalty on out-of-distribution actions

## DrQ-v2 Loss Functions

**Critic Loss** (TD error):
```python
# File: /home/gota/ggando/ml/lerobot/src/lerobot/policies/drqv2/modeling_drqv2.py:851
td_target = rewards + discount * (1 - done) * min_target_q
critic_loss = MSE(q_values, td_target)
```

**Actor Loss** (maximize Q):
```python
# File: /home/gota/ggando/ml/lerobot/src/lerobot/policies/drqv2/modeling_drqv2.py:903
min_q = q_values.min(dim=-1)[0]
actor_loss = -min_q.mean()  # Maximize Q-values
```

## Why Sparse Rewards are Problematic

1. **Credit Assignment**: 100+ steps before reward, unclear which actions contributed
2. **Exploration**: Random actions rarely reach goal state
3. **Sample Efficiency**: Most transitions have zero reward
4. **Q-function Learning**: TD target is mostly `0.99 * next_Q`, propagating noise

**Demonstration Data Helps But Not Enough**:
- Offline buffer: 793 transitions, 18 rewards (2.3%)
- Online buffer with intervention: 37% intervention rate
- But still only 0.9% of all data has positive reward

## Intervention Action Recording

Verified correct in `actor.py:586-590`:
```python
if "is_intervention" in info and info["is_intervention"]:
    action = info["action_intervention"]  # Use human action
    episode_intervention = True
```

The intervention actions are recorded correctly, but:
- Only intervention **during episode** is recorded
- Human provides demonstrations, but reward is still sparse
- No explicit behavior cloning loss to directly learn from demos

## Robot Behavior: "Shaking"

**Symptom**: Robot exhibits shaking/oscillating behavior instead of smooth motions.

**Likely Causes**:
1. Exploration noise too high: `stddev_schedule: linear(0.2,0.05,5000)` starts at 0.2
2. Q-values unstable: overestimation leads to rapidly changing gradients
3. No temporal smoothing: each timestep action is independent
4. Conflicting gradients: maximizing overestimated Q vs learning from sparse rewards

## Potential Solutions (Not Yet Implemented)

### 1. Add Behavior Cloning Loss
```python
# Actor loss with BC regularization
bc_loss = MSE(actor_actions, demo_actions)  # On demo data
actor_loss = -min_q.mean() + bc_weight * bc_loss
```
**Status**: No BC support in current DrQ-v2 implementation

### 2. Conservative Q-Learning (CQL)
```python
# Penalize Q-values on out-of-distribution actions
cql_loss = logsumexp(Q(s, random_actions)) - E[Q(s, demo_actions)]
```
**Status**: Would require modifying critic loss

### 3. Reward Shaping
- Add dense progress reward toward cube
- Penalize distance from demonstration trajectory
**Status**: Would require modifying reward function in environment

### 4. Higher UTD Ratio
- More gradient updates per environment step
- Helps with sample efficiency
**Status**: Proposed but rejected by user

### 5. Pre-training with BC
- Initialize policy with pure behavior cloning
- Then fine-tune with RL
**Status**: Would require separate BC training script

## Files Reference

| File | Purpose |
|------|---------|
| `lerobot/scripts/rl/learner.py` | Training loop, buffer management |
| `lerobot/scripts/rl/actor.py` | Environment interaction, intervention handling |
| `lerobot/policies/drqv2/modeling_drqv2.py` | DrQ-v2 policy, loss computation |
| `lerobot/scripts/rl/gym_manipulator.py` | Environment wrappers |
| `outputs/hilserl_drqv2_scratch/train_config.json` | Training configuration |

## Critical Finding: No Demonstration-Specific Handling

**The DrQ-v2 implementation has NO special handling for demonstrations/interventions.**

Verified via code search:
```bash
grep -i "intervention|demo|bc_|behavior_cloning" modeling_drqv2.py
# No matches found
```

### What HIL-SERL Typically Requires

Standard HIL-SERL (with SAC) includes:
1. **Demo relabeling** - Propagate rewards back through demo trajectories
2. **Priority sampling** - Higher probability of sampling demo transitions
3. **BC loss component** - Direct imitation learning on demonstrations
4. **Critic filtering** - Only update critic on high-value transitions

### What This Implementation Has

This DrQ-v2 implementation only:
1. Adds intervention transitions to both online and offline buffers (line 1485 in learner.py)
2. Samples 50% from each buffer during training
3. Uses standard TD-learning on all transitions equally

**No explicit mechanism to learn from demonstrations directly.**

## Why ~50 Episodes Isn't Enough

In standard HIL-SERL (SAC + demo handling):
- 50 successful episodes ≈ 5000+ high-quality transitions
- BC loss directly updates actor to match demonstrations
- Demo priority sampling ensures high replay frequency

In this DrQ-v2 implementation:
- 50 successful episodes = ~50 positive reward transitions
- No BC loss - only TD-learning
- Sparse terminal reward provides almost no gradient signal
- Intervention actions help critic but don't directly teach actor

**The intervention data is stored correctly but not used effectively.**

## Summary

Training DrQ-v2 from scratch with HIL-SERL faces fundamental challenges:

1. **Sparse rewards** (0.9% positive reward rate) make Q-learning difficult
2. **Q-value overestimation** (Q > 1.0 when max reward is 1.0) destabilizes training
3. **No BC loss** means intervention data only helps via Q-function, not direct imitation
4. **Robot shaking** results from noisy Q-gradients and high exploration noise

The HIL-SERL framework stores intervention actions but relies on TD-learning to extract signal from them. With sparse terminal rewards, this is insufficient for learning from scratch.

**Root Cause**: The DrQ-v2 policy lacks demonstration-specific mechanisms that make HIL-SERL effective. It needs:
- Behavior cloning loss on demonstration data (learn to directly imitate)
- Or: Use a policy that already knows roughly what to do (sim-pretrained)

## Fix Implemented: BC Loss Added to DrQ-v2

Added behavior cloning loss to the actor update in `modeling_drqv2.py`:

```python
# In compute_loss_actor():
actor_loss = -min_q.mean()

# Add behavior cloning loss for HIL-SERL
bc_loss = torch.tensor(0.0, device=actor_loss.device)
if self.config.bc_weight > 0:
    complementary_info = batch.get("complementary_info")
    if complementary_info is not None and "is_intervention" in complementary_info:
        intervention_mask = complementary_info["is_intervention"]
        if intervention_mask.dtype != torch.bool:
            intervention_mask = intervention_mask.bool()

        if intervention_mask.any():
            demo_actions = batch["action"][intervention_mask]
            actor_actions = actions[intervention_mask]
            bc_loss = F.mse_loss(actor_actions, demo_actions)
            actor_loss = actor_loss + self.config.bc_weight * bc_loss
```

### Configuration

Added `bc_weight` to `configuration_drqv2.py`:
```python
# Behavior cloning weight for HIL-SERL (demonstration learning)
bc_weight: float = 0.0  # Default off for backwards compatibility
```

Updated `train_config.json`:
```json
"bc_weight": 0.5
```

### How It Works

1. During online training, intervention transitions have `is_intervention=True` in `complementary_info`
2. When actor loss is computed, BC loss is added for intervention samples
3. Actor learns both from:
   - RL signal (maximize Q-values)
   - BC signal (directly imitate demonstrations)

### Limitations

- BC loss only applies to intervention transitions from online buffer
- Original offline dataset doesn't have `is_intervention` flag
- Intervention transitions get added to both online and offline buffers, so they're sampled more frequently

### Files Modified

1. `lerobot/src/lerobot/policies/drqv2/configuration_drqv2.py`:
   - Added `bc_weight: float = 0.0` config parameter

2. `lerobot/src/lerobot/policies/drqv2/modeling_drqv2.py`:
   - Modified `compute_loss_actor()` to add BC loss on intervention samples

3. `lerobot/src/lerobot/scripts/rl/learner.py`:
   - Added `complementary_info` to `forward_batch` (was missing, so BC loss would never trigger)

4. `outputs/hilserl_drqv2_scratch/train_config.json`:
   - Set `bc_weight: 0.5`

### Testing

```bash
# Verify imports work
cd /home/gota/ggando/ml/lerobot
python -c "from lerobot.policies.drqv2.modeling_drqv2 import DrQV2Policy; print('OK')"
python -c "from lerobot.scripts.rl.learner import add_actor_information_and_train; print('OK')"

# Verify config
python -c "import json; c = json.load(open('../so101-playground/outputs/hilserl_drqv2_scratch/train_config.json')); print('bc_weight:', c['policy'].get('bc_weight'))"
# Output: bc_weight: 0.5
```

### Next Steps

To restart training with BC loss enabled:
1. Optionally delete checkpoint to start fresh: `rm -rf outputs/hilserl_drqv2_scratch/checkpoints`
2. Delete offline buffer cache to regenerate: `rm outputs/hilserl_drqv2_scratch/offline_buffer.pt`
3. Or continue training - BC loss will apply to new intervention data

### Expected Behavior

With `bc_weight=0.5`, during actor update:
- Standard RL loss: `-min_q.mean()` (maximize Q-values)
- BC loss: `0.5 * MSE(actor_actions, demo_actions)` on intervention samples
- Combined loss makes actor directly imitate demonstrations while also optimizing Q

This should significantly improve learning from interventions, especially with sparse rewards where Q-learning alone struggles.
