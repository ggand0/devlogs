# Devlog 059: BC Loss Experiment (Reverted)

Documenting an experimental BC loss addition to DrQ-v2 that was implemented but reverted due to lack of evidence it would work for this specific use case.

## Context

Training DrQ-v2 from scratch with HIL-SERL was not learning after ~50 intervention episodes. Robot exhibited shaking behavior.

## Hypothesis (Unverified)

BC loss would help the actor directly learn from intervention demonstrations instead of only through Q-function gradients.

## Source of BC Loss Idea

From general RL+IL literature in training data (not verified for this specific case):
- DDPGfD (Vecerik et al. 2017) - adds BC loss for sparse reward manipulation
- DAPG (Rajeswaran et al. 2018) - combines policy gradient with BC
- AWAC (Nair et al. 2020) - implicit BC through advantage weighting

**Problem**: No evidence these approaches work with DrQ-v2 + HIL-SERL specifically.

## What HIL-SERL Actually Uses

From the [HIL-SERL paper](https://arxiv.org/html/2410.21845v1):
- **Algorithm**: RLPD (not DrQ-v2)
- **NO BC loss** in actor update
- **Reward classifier** provides reward at every timestep (binary 0/1)
- Demos only in replay buffer, not in loss function

The key difference: HIL-SERL's "sparse" reward = classifier prediction every step, not terminal-only reward.

## Code Changes Made (Now Reverted)

### 1. configuration_drqv2.py

Added after `skip_pretrained_critic`:
```python
# Behavior cloning weight for HIL-SERL (demonstration learning)
# When > 0, adds BC loss on intervention transitions to help actor learn from demonstrations
# Typical values: 0.1-1.0. Set to 0 to disable BC loss.
bc_weight: float = 0.0
```

### 2. modeling_drqv2.py

Modified `compute_loss_actor()`:
```python
def compute_loss_actor(self, batch: dict[str, torch.Tensor]) -> torch.Tensor:
    """Compute actor loss using policy gradient with optional behavior cloning.

    DrQ-v2 actor loss: -min(Q(s, pi(s))) + bc_weight * BC_loss

    When bc_weight > 0 and intervention data is present, adds a behavior cloning
    loss to directly learn from demonstration actions (HIL-SERL support).
    """
    # ... existing code ...

    # Actor loss: maximize Q (minimize -Q)
    min_q = q_values.min(dim=-1)[0]  # (B,)
    actor_loss = -min_q.mean()

    # Add behavior cloning loss for HIL-SERL
    bc_loss = torch.tensor(0.0, device=actor_loss.device)
    if self.config.bc_weight > 0:
        complementary_info = batch.get("complementary_info")
        if complementary_info is not None and "is_intervention" in complementary_info:
            intervention_mask = complementary_info["is_intervention"]
            # Handle both bool tensor and float tensor
            if intervention_mask.dtype != torch.bool:
                intervention_mask = intervention_mask.bool()

            if intervention_mask.any():
                # Get demonstration actions from batch
                demo_actions = batch["action"][intervention_mask]
                # Get actor's actions for same states
                actor_actions = actions[intervention_mask]
                # MSE loss between actor and demonstration actions
                bc_loss = F.mse_loss(actor_actions, demo_actions)
                actor_loss = actor_loss + self.config.bc_weight * bc_loss

    return actor_loss
```

### 3. learner.py

Added `complementary_info` to `forward_batch`:
```python
forward_batch = {
    "action": actions,
    "reward": rewards,
    "state": observations,
    "next_state": next_observations,
    "done": done,
    "observation_feature": observation_features,
    "next_observation_feature": next_observation_features,
    "complementary_info": batch.get("complementary_info"),  # Added
}
```

### 4. train_config.json

Set `bc_weight: 0.5`

## Why Reverted

1. **No evidence** BC loss works with DrQ-v2 + HIL-SERL
2. **Original HIL-SERL doesn't use BC** - relies on per-step reward classifier
3. **Different reward structure**: HIL-SERL has reward every step, we have terminal-only
4. Suggested based on general RL intuition, not specific validation

## Actual Solutions (From HIL-SERL Paper)

1. **Train a reward classifier** - provides dense per-step reward signal
2. **Use sim-pretrained policy** - already knows roughly what to do
3. **Use RLPD algorithm** - designed specifically for prior data

## Lessons Learned

- Don't implement fixes based on general intuition without evidence for specific setup
- HIL-SERL works because of reward classifier, not because of demo handling in loss
- The "sparse reward" in HIL-SERL is actually dense (every step) compared to terminal-only

---

## Update: BC Pretraining for SAC (2026-01-26)

Instead of adding BC loss during RL training, implemented a separate BC pretraining script to initialize the SAC actor before RL begins.

### Approach

1. **Separate pretraining phase**: Train actor with supervised learning on demos
2. **Then RL fine-tuning**: Start HIL-SERL with pretrained weights
3. **No BC loss during RL**: Pure SAC after pretraining

This is different from the reverted approach (BC loss during training) and aligns with standard practice in robotics RL.

### Implementation

Script: `scripts/bc_pretrain_sac.py`

Key features:
- Loads demo dataset and SAC policy
- Freezes critic, only trains actor (and encoder if not frozen)
- MSE loss between actor output and demo actions
- Saves pretrained model in HuggingFace format

### Bug Fix

Initial version had a critical bug - multiplying images by 255:
```python
# BUGGY (removed):
if obs.max() <= 1.0:
    obs = obs * 255.0  # WRONG!
```

Dataset stores images in [0, 1], policy expects [0, 1]. The 255x scaling caused distribution mismatch.

### Training Results (2026-01-26)

| Metric | Value |
|--------|-------|
| Steps | 3000 |
| Epochs | 375 |
| Final Loss | **0.0065** |
| Training Time | 2h 23min |
| Dataset | 2220 samples |

Loss progression:
```
Step 100:  0.0915
Step 500:  0.0231
Step 1000: 0.0117
Step 2000: 0.0083
Step 3000: 0.0065
```

### Output

- BC model: `outputs/hilserl_reach_grasp_v4_bc_pretrained/bc_pretrained/final/pretrained_model`
- Checkpoint: `outputs/hilserl_reach_grasp_v4_bc_pretrained/checkpoints/000000/pretrained_model`

### Loading into HIL-SERL

Since `pretrained_path` is not a valid SACConfig field, used checkpoint workaround:
1. Created `checkpoints/000000/pretrained_model/` with BC weights
2. Set `resume: true` in config
3. Learner loads BC weights on start

### Expected Benefit

- Actor should already know basic reaching behavior from BC
- RL fine-tuning focuses on grasping and edge cases
- Lower intervention rate expected compared to training from scratch
