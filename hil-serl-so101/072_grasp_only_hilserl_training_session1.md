# Grasp-Only HIL-SERL Training Session 1

**Date**: 2026-01-26
**Task**: Grasp cube from random positions (grasp-only, no reach phase)

## Training Configuration

| Parameter | Value |
|-----------|-------|
| Checkpoint | Resumed from step 10000 |
| Episodes | ~527 |
| Steps | ~11000 (stopped at 11000, resumed from 10000) |
| Episode length | 8 seconds @ 10fps (~80 steps/episode) |
| UTD ratio | 20 |
| Batch size | 256 |
| Discount | 0.97 |
| Temperature init | 0.01 |
| Buffer capacity | 30000 (online), 5000 (offline) |

## Reset Configuration

- IK reset enabled with random offset
- Base EE position: [0.25, 0.0, 0.03]
- Random XY range: ±3cm
- Random Z range: ±2cm
- Reset delay: 2.0s

## Reward Configuration

- Reward classifier: `reward_classifier_grasp_only_v1`
- Threshold: 0.7 (hardcoded)
- Min steps before success: 30 (3 seconds hold)
- Consecutive success frames: 10

## Training Metrics (Step 10900-11100)

```
Step 10900: critic=235.2459 actor=-103.2764 temp=-0.0773 α=0.1534 buffer=30000
Step 11000: critic=87.1461  actor=-102.1810 temp=-0.0460 α=0.1559 buffer=30000
Step 11100: critic=106.7973 actor=-103.7143 temp=-0.0373 α=0.1574 buffer=30000
```

- Critic loss fluctuating (87-235) - expected for value learning
- Actor loss stable around -102 to -103
- Temperature α ~0.15 (exploring but not too random)
- Buffer at full capacity (30000)

## Observations

### What's Working
- Policy learned to grasp from **easy cube positions/rotations**
- Reward classifier detecting grasps reliably
- Random EE reset providing position variation
- Human interventions being applied as per HIL-SERL guidelines

### What's Not Working
- Policy struggles with **harder cube positions/rotations**
- Not yet generalizing across full workspace

### Intervention Strategy Applied
- Intervening frequently early (every episode or every other episode)
- Helping policy get reward in ~1/3+ of episodes
- Letting policy explore then correcting when stuck

## HIL-SERL Paper Reference

From the paper (arxiv 2410.21845):
- Training times: 1-2.5 hours for most tasks
- Paper reports **only wall-clock time**, not step counts
- Figure 4 shows learning curves as "running average over 20 episodes"
- No specific step/episode convergence numbers provided

## Next Steps

1. Continue training with more interventions on hard positions
2. Use interventions to specifically practice difficult cube orientations
3. Monitor if policy generalizes as training continues
4. Consider collecting more reward classifier negatives if false positives appear

---

## Session 2: Continuation (Episodes 164-203)

**Resumed from**: Step 10000
**Ended at**: Step ~14076, Episode 203

### Training Metrics (Step 12600-13200)

```
Step 12600: critic=87.1658  actor=-90.6946 temp=0.0788 α=0.1694 buffer=11305
Step 12700: critic=69.2468  actor=-96.0243 temp=0.0220 α=0.1667 buffer=11734
Step 12800: critic=129.2553 actor=-88.2870 temp=-0.0250 α=0.1648 buffer=12185
Step 12900: critic=194.0995 actor=-82.8147 temp=-0.0424 α=0.1630 buffer=12621
Step 13000: critic=118.2112 actor=-86.8628 temp=0.0673 α=0.1611 buffer=13014
Step 13100: critic=119.6432 actor=-89.2170 temp=0.0041 α=0.1599 buffer=13472
Step 13200: critic=199.9256 actor=-84.5903 temp=0.0161 α=0.1600 buffer=13917
```

### Episode Results (Sample)

| Episode | Step | Reward | Intervention | Notes |
|---------|------|--------|--------------|-------|
| 164 | 11147 | -4.00 | 0.0% | Failed (hard position) |
| 165 | 11227 | 16.10 | 0.0% | Low reward |
| 166 | 11304 | 177.05 | 53.2% | Success with intervention |
| 168 | 11416 | 179.30 | 0.0% | Success (easy position) |
| 169-170 | - | -4.00 | 0.0% | Failed |
| 174 | 11884 | 138.20 | 0.0% | Success (easy position) |
| 176 | 12024 | 208.05 | 96.7% | Success with heavy intervention |
| 185 | 12700 | 76.40 | 0.0% | Success |
| 192-196 | - | -4.00 | 0.0% | 5 consecutive failures |
| 199 | 13756 | 118.90 | 0.0% | Success (easy position) |
| 200-201 | - | -4.00 | 0.0% | Failed |

### Pattern Identified: Local Optimum

**Problem**: Policy stuck in local optimum
- Learned a fixed motion pattern that works for **cubes in the back** (easy positions)
- Same motion fails for **cubes in the front** (hard positions)
- Despite interventions, policy keeps reverting to the easy-position motion
- Many episodes with reward=-4.00 are hard positions where policy doesn't adapt

**Observed Behavior**:
- Policy executes the same grasp motion regardless of cube position
- "Hopes" the cube is in an easy position
- Doesn't adjust trajectory based on visual input for different positions

### Intervention Analysis

Episodes with high intervention (>50%):
- Episode 166: 53.2% → reward 177.05
- Episode 172: 96.1% → reward 7.25
- Episode 176: 96.7% → reward 208.05
- Episode 179: 60.0% → reward -4.00 (still failed)
- Episode 180: 89.7% → reward -2.35
- Episode 191: 91.4% → reward 117.70

**Issue**: Even with heavy intervention, policy doesn't learn from the corrections. The interventions help complete individual episodes but don't transfer to policy behavior.

### Possible Causes

1. **Visual encoder not learning position-dependent features** - frozen ResNet10 may not extract cube position well
2. **State representation insufficient** - 18-dim proprioception may not encode enough spatial info
3. **Reward signal too sparse** - only terminal reward, no intermediate feedback
4. **Action space too coarse** - 0.02 action_scale may limit fine adjustments
5. **Distribution shift** - easy positions overrepresented in successful transitions

### Potential Solutions to Explore

1. **Curriculum**: Start with smaller random reset range, gradually increase
2. **Dense reward**: Add distance-based shaping (but we removed this earlier)
3. **More demonstrations**: Collect demos specifically from hard positions
4. **Unfreeze encoder**: Allow vision encoder to fine-tune
5. **Increase exploration**: Higher temperature to escape local optimum

## Session Status

**Paused** at step ~14076, Episode 203
**Checkpoint saved**: 013000

**Verdict**: Policy stuck in local optimum. 200 additional episodes with interventions did not improve generalization to hard cube positions.

---

## HIL-SERL Parameter Analysis

### Comparison with Original Implementation

Analyzed original HIL-SERL code (`hil-serl/serl_launcher/`) to identify parameter differences.

| Parameter | HIL-SERL Original | Our Config | Status |
|-----------|-------------------|------------|--------|
| `temperature_init` | 0.01 | 0.01 | ✓ |
| `discount` | 0.97 | 0.97 | ✓ |
| `backup_entropy` | False | ~~True~~ → False | Fixed |
| `critic_ensemble_size` | 2 | 2 | ✓ |
| `learning_rate` (all) | 3e-4 | 3e-4 | ✓ |
| `soft_target_update_rate` | 0.005 | 0.005 | ✓ |
| `target_entropy` | -action_dim/2 | -2.0 | ✓ |
| `replay_buffer_capacity` | 200,000 | ~~30,000~~ → 200,000 | Fixed |
| `cta_ratio` / `utd_ratio` | 2 | 20 | OK (higher is fine) |
| **Data augmentation** | `batched_random_crop(pad=4)` | None | **Missing** |

### Key Finding: Data Augmentation

HIL-SERL uses random crop augmentation during training (`launcher.py:46`):
```python
augmentation_function=make_batch_augmentation_func(image_keys)
```

This applies `batched_random_crop(padding=4)` to both observations and next_observations in each batch.

**Our SAC implementation has NO data augmentation** - this could explain the local optimum:
- Without augmentation, network can memorize exact pixel patterns
- Frozen encoder + no augmentation = very limited position generalization

### Config Changes Made

1. `use_backup_entropy`: true → false
2. `online_buffer_capacity`: 30,000 → 200,000

### Next Steps

1. Add random shift augmentation to SAC policy (port from DrQ-v2)
2. Or switch to DrQ-v2 which has built-in `RandomShiftsAug(pad=4)`

---

## Encoder Sensitivity Test

Tested whether frozen ResNet10 encoder can distinguish cube positions (front vs back).

### Test Setup
- Captured gripper cam with cube in front position: `outputs/gripper_cam_current.png`
- Captured gripper cam with cube in back position: `outputs/gripper_cam_back.png`
- Preprocessed with same crop [0:480, 80:560] and resize to 128x128
- Fed through `helper2424/resnet10` encoder

### Results

| Metric | Value |
|--------|-------|
| Cosine similarity | 0.6294 |
| L2 distance | 34.22 |
| Embedding dim | 512 |

### Conclusion

**The frozen encoder CAN distinguish cube positions.** 0.63 cosine similarity indicates meaningful difference between front and back positions.

The encoder is NOT the bottleneck. The issue is likely:
- Policy/critic networks not learning to use the position information
- Missing data augmentation (now added: `use_augmentation: true`)
- Other training dynamics

HIL-SERL also uses frozen encoder with `jax.lax.stop_gradient()` in `resnet_v1.py:288`, but trains the spatial learned embeddings layer on top.

---

## HIL-SERL Best Practices (from official docs)

### Intervention Strategy

1. **Early training**: Intervene frequently (every episode or every other episode)
2. **Let policy explore first**: Allow 20-30 timesteps of exploration, THEN guide toward task region
3. **Reward frequency**: Help policy get reward in ~1/3+ of episodes
4. **Reduce over time**: Intervention rate should drop as policy improves
5. **Engineer edge cases**: Deliberately practice failure scenarios to build recovery behaviors

### Temperature

- `temperature_init = 0.01` is recommended starting point
- **Higher temperature makes human interventions LESS effective** and slows learning

### Reward Classifier

- Collect **2-3x more negatives** than positives
- When false positives appear, **collect more data** targeting failure modes before adjusting thresholds
- 20 demonstrations is standard

### Task Design

- Tasks should complete in **5-10 seconds**
- Avoid long horizon tasks initially

### Expected Training Times (Franka)

- RAM insertion: ~1.5 hours
- USB pickup/insertion: ~2.5 hours

### Current Session Analysis

**What we may be doing wrong:**
- Not letting policy explore enough before intervening?
- Not engineering edge cases (hard cube positions) deliberately?
- Intervention rate not dropping = policy not learning from corrections

**Potential fixes:**
1. Let policy run 20-30 steps before intervening
2. When intervening, do quick corrective actions, don't take over for long periods
3. Deliberately practice hard cube positions with interventions
4. Collect more demos from hard positions for offline buffer

---

## HIL-SERL Original Hyperparameters (from launcher.py)

```python
temperature_init = 1e-2  # 0.01
discount = 0.97
backup_entropy = False
critic_ensemble_size = 2
critic_subsample_size = None
hidden_dims = [256, 256]  # both actor and critic
activations = nn.tanh
use_layer_norm = True
soft_target_update_rate = 0.005
learning_rate = 3e-4  # all networks
```

Data augmentation: `batched_random_crop` applied to **ALL image keys** on EVERY batch.

---

## Critical Paper Insights (arxiv 2410.21845)

### Intervention Anti-Pattern

**AVOID**: "Persistently providing long sparse interventions that lead to task successes"

**WHY**: This causes "overestimation of the value function, particularly in the early stages... resulting in unstable training dynamics."

**INSTEAD**: Keep interventions **brief and frequent** initially.

### Data Requirements

| Data Type | Amount |
|-----------|--------|
| Reward classifier positives | ~200 examples |
| Reward classifier negatives | ~1000 examples (5x positives) |
| Offline demonstrations | 20-30 trajectories |
| Classifier accuracy before RL | >95% |

### Value Function Overestimation Fix

The paper stores interventions in BOTH demo and RL buffers, but:
- "The policy's transitions (i.e., states and actions **before and after** the intervention) only to the RL buffer"
- This prevents overestimating value from consistently successful human takeovers

### Task-Specific Findings

| Task | Training Time | Notes |
|------|---------------|-------|
| RAM insertion | ~1.5 hours | Wrist-mounted cameras, ego-centric |
| USB pickup/insertion | ~2.5 hours | - |
| Jenga whipping | - | 30 offline demos (no real-time corrections due to dynamics) |
| IKEA assembly | - | Chain subtasks, ±1cm pose randomization |

### Common Failure Modes

| Failure | Solution |
|---------|----------|
| Poor classifier | Collect more false-positive/false-negative examples; >95% accuracy |
| Learning instability | Use pretrained vision (ResNet-10 ImageNet) |
| Value overestimation | Brief frequent interventions, not long sparse ones |
| Poor spatial generalization | Ego-centric coordinates, randomize EE initial pose |

---

## Our Config vs HIL-SERL

| Parameter | HIL-SERL | Ours | Match |
|-----------|----------|------|-------|
| temperature_init | 0.01 | 0.01 | ✓ |
| discount | 0.97 | 0.97 | ✓ |
| backup_entropy | False | False | ✓ |
| critic_ensemble_size | 2 | 2 | ✓ |
| soft_target_update_rate | 0.005 | 0.005 | ✓ |
| learning_rate | 3e-4 | 3e-4 | ✓ |
| hidden_dims | [256, 256] | [256, 256] | ✓ |
| use_layer_norm | True | ? | Check |
| data augmentation | batched_random_crop | RandomShiftsAug | ✓ (similar) |
| buffer capacity | 200,000 | 200,000 | ✓ |

### Layer Norm

HIL-SERL uses `use_layer_norm=True` in both actor and critic networks. Our SAC implementation also uses `nn.LayerNorm` in:
- Encoder output (line 604, 615, 622)
- MLP layers (line 767)

✓ Layer norm matches.

---

## Session 3 Action Items

Based on this research, for the next training session:

1. **Change intervention style**: Let policy explore 20-30 steps, then do BRIEF corrections (not long takeovers)
2. **Watch for value overestimation**: If policy seems to rely on interventions, reduce intervention frequency
3. **Collect more demos from hard positions**: Add 10-20 demos specifically from front/hard cube positions to offline buffer
4. **Verify reward classifier**: Check for false positives/negatives during training
5. **Track intervention rate over time**: Should decrease as policy improves; if not, something is wrong
