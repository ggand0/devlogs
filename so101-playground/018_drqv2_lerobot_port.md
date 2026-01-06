# Devlog 018: DrQ-v2 Policy Port to LeRobot

## Date: 2026-01-06

## Summary

Implemented DrQ-v2 policy in LeRobot to enable fine-tuning of sim-trained RoboBase checkpoints on real SO-101 robot with HIL-SERL workflow.

## Problem

- HIL-SERL in LeRobot uses SAC, which trains from scratch
- Sim-trained DrQ-v2 policy (70% success in MuJoCo) uses RoboBase codebase
- Architecture mismatch between RoboBase DrQ-v2 and LeRobot SAC:
  - Different encoder structure (spatial embeddings vs simple CNN)
  - No random shift augmentation in LeRobot
  - Different checkpoint formats (.pt vs safetensors)

## Solution

Ported DrQ-v2 to LeRobot matching RoboBase architecture exactly.

## Implementation Details

### Files Created (in LeRobot fork)

```
lerobot/src/lerobot/policies/drqv2/
├── __init__.py
├── configuration_drqv2.py      # DrQV2Config with RoboBase-compatible params
└── modeling_drqv2.py           # Full implementation

lerobot/scripts/
└── convert_robobase_checkpoint.py  # Checkpoint converter
```

### Key Components

#### 1. RandomShiftsAug (lines 117-142)
Key DrQ-v2 regularization technique:
- Pads image with replicate padding (pad=4)
- Randomly shifts via `F.grid_sample`
- Only applied during training

```python
class RandomShiftsAug(nn.Module):
    def __init__(self, pad: int = 4):
        self.pad = pad

    def forward(self, x):
        # Replicate padding + random crop
        # Uses grid_sample for differentiable shifting
```

#### 2. DrQV2Encoder (lines 150-220)
Matches `EncoderCNNMultiViewDownsampleWithStrides`:
- Input normalization: `x / 255.0 - 0.5` (NOT ImageNet mean/std)
- 1 downsample conv (stride=2) + 3 post-downsample convs (stride=1)
- 32 channels, 3x3 kernels, no padding
- Output: 39200 features for 84x84 input

#### 3. MLPWithBottleneck (lines 260-340)
Actor/critic network architecture:
- Bottleneck: `Linear(input, 50) → LayerNorm → Tanh`
- Main MLP: `Linear → Identity → ReLU` for each hidden layer [256, 256]
- Output: Linear to action_dim or 1

#### 4. Weight Initialization
Uses RoboBase's `weight_init`:
- Linear: orthogonal init
- Conv2d: orthogonal with relu gain
- LayerNorm: weight=1, bias=0

### Checkpoint Loading

`DrQV2Policy.from_robobase_checkpoint()` infers all dimensions from weights:
```python
# Inferred from sim checkpoint:
- Input channels: 9 (frame_stack=3)
- Encoder channels: 32
- Fused feature dim: 39200
- Low-dim state size: 63
- Action dim: 4
- Bottleneck size: 50
- MLP nodes: [256, 256]
```

### Weight Mapping

```
RoboBase Key                                    → LeRobot Key
encoder.convs_per_cam.0.*                       → encoder.convs_per_cam.0.*
actor.actor_model.input_preprocess_modules.*    → actor.actor_model.input_preprocess_modules.*
actor.actor_model.main_mlp.*                    → actor.actor_model.main_mlp.*
actor.actor_model.out_mlp.*                     → actor.actor_model.out_mlp.*
critic.qs.0.*                                   → critic.qs.0.*
critic_target.qs.0.*                            → critic_target.qs.0.*
```

Skipped keys: `*hidden_state` (RNN buffers, not trained weights)

## Usage

### Load from RoboBase checkpoint
```python
from lerobot.policies.drqv2 import DrQV2Policy

policy = DrQV2Policy.from_robobase_checkpoint(
    '/path/to/robobase/checkpoint.pt',
    device='cuda'
)
```

### Convert checkpoint to safetensors
```bash
python scripts/convert_robobase_checkpoint.py \
    --input /path/to/robobase/checkpoint.pt \
    --output /path/to/lerobot/checkpoint
```

## Verified Working

- [x] DrQV2Config imports and instantiates
- [x] DrQV2Policy imports
- [x] Checkpoint converter extracts 92/98 keys (skips 6 hidden states)
- [x] `from_robobase_checkpoint()` infers correct dimensions
- [x] Inference produces valid actions: `[-1.0, -0.99, 1.0, 1.0]`
- [x] Full training loop with critic/actor/temperature forward passes
- [x] Gradients flow correctly through encoder (via critic), detached for actor
- [x] Compatible with learner.py interface

## Source Checkpoint

```
pick-101/runs/image_rl/20260101_125922/snapshots/latest_snapshot.pt
```

Config from checkpoint:
- stddev_schedule: `linear(1.0,0.1,500000)`
- stddev_clip: 0.3
- use_augmentation: true
- critic_target_tau: 0.01
- discount: 0.99

## Training Loop Implementation

### forward() Method
The policy now supports the learner.py interface with separate model updates:
- `forward(batch, model="critic")`: TD learning with augmentation
- `forward(batch, model="actor")`: Policy gradient (detached encoder)
- `forward(batch, model="temperature")`: No-op for DrQ-v2 (returns 0 loss)

### Key Training Differences from SAC
| Feature | SAC | DrQ-v2 |
|---------|-----|--------|
| Actor loss | `(temperature * log_prob) - min_q` | `-min_q` |
| Exploration | Entropy-based | Scheduled noise stddev |
| Temperature | Learned log_alpha | Not used |
| Augmentation | None | RandomShiftsAug during training |

### learner.py Compatibility
Added compatibility attributes to DrQV2Config:
- `num_discrete_actions = None`
- `shared_encoder = True`
- `vision_encoder_name = None`
- `freeze_vision_encoder = False`

Added to DrQV2Policy:
- `critic_ensemble` (alias to `critic`)
- `log_alpha` (dummy param for optimizer compatibility)
- `update_temperature()` (no-op)

## Next Steps

1. ~~**Implement full training loop** in `forward()`~~ ✅

2. **Record demonstrations**:
   - Need human demos to bootstrap replay buffer
   - Sim policy completely fails on real robot without fine-tuning

3. **Test HIL-SERL training**:
   - Run learner.py with DrQ-v2 config
   - Verify training metrics look correct
   - Test on real SO-101 with human interventions

## Files Modified

### so101-playground
- `devlogs/018_drqv2_lerobot_port.md` (this file)
- `docs/plans/drqv2-lerobot-port.md` (implementation plan)

### LeRobot fork (branch: feat/drqv2-policy)
- `src/lerobot/policies/drqv2/__init__.py`
- `src/lerobot/policies/drqv2/configuration_drqv2.py`
- `src/lerobot/policies/drqv2/modeling_drqv2.py`
- `src/lerobot/policies/__init__.py` (added DrQV2Config)
- `src/lerobot/policies/factory.py` (added drqv2 support)
- `scripts/convert_robobase_checkpoint.py`
- `pyproject.toml` (added omegaconf dependency)

## References

- RoboBase DrQ-v2: `/home/gota/ggando/ml/pick-101/external/robobase/robobase/method/drqv2.py`
- RoboBase encoder: `/home/gota/ggando/ml/pick-101/external/robobase/robobase/models/encoder.py`
- LeRobot SAC: `/home/gota/ggando/ml/lerobot/src/lerobot/policies/sac/`
