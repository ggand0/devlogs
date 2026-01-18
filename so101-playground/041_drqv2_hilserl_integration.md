# Devlog 041: DrQ-v2 HIL-SERL Integration

**Date**: 2025-01-18
**Status**: Complete

## Overview

Integrated DrQ-v2 (Data-regularized Q-learning version 2) policy with HIL-SERL distributed training infrastructure. DrQ-v2 is an alternative to SAC that uses random shift augmentation instead of entropy regularization for exploration.

## Background

The existing HIL-SERL setup uses SAC with a pretrained ResNet encoder. DrQ-v2 offers:
- Custom CNN encoder trained end-to-end
- Random shift augmentation as regularization
- Scheduled exploration noise (vs entropy-based)
- Simpler architecture with fewer hyperparameters

## Changes Made

### 1. Learner Encoder Optimization

**File**: `lerobot/src/lerobot/scripts/rl/learner.py`

DrQ-v2's encoder is separate from the critic and needs its own optimizer. Added encoder optimizer support:

```python
# In make_optimizers_and_scheduler()
if is_drqv2:
    encoder_lr = getattr(cfg.policy, "encoder_lr", cfg.policy.critic_lr)
    optimizers["encoder"] = torch.optim.Adam(
        params=policy.encoder.parameters(), lr=encoder_lr
    )
```

Added encoder gradient handling in both UTD ratio loop and main critic optimization:

```python
# Zero encoder gradients before critic backward
if "encoder" in optimizers:
    optimizers["encoder"].zero_grad()
loss_critic.backward()

# Step encoder optimizer after critic
if "encoder" in optimizers:
    torch.nn.utils.clip_grad_norm_(
        parameters=policy.encoder.parameters(), max_norm=clip_grad_norm_value
    )
    optimizers["encoder"].step()
```

### 2. DrQ-v2 Config Compatibility

**File**: `lerobot/src/lerobot/policies/drqv2/configuration_drqv2.py`

Added fields and dataclasses for learner compatibility:

```python
# New dataclasses
@dataclass
class ConcurrencyConfig:
    actor: str = "threads"
    learner: str = "threads"

@dataclass
class ActorLearnerConfig:
    learner_host: str = "127.0.0.1"
    learner_port: int = 50051
    policy_parameters_push_frequency: int = 4
    queue_get_timeout: float = 2

# New fields in DrQV2Config
utd_ratio: int = 1  # Update-to-data ratio
async_prefetch: bool = False  # Replay buffer async prefetch
offline_buffer_capacity: int = 50000  # Demonstration buffer capacity
actor_learner_config: ActorLearnerConfig = field(default_factory=ActorLearnerConfig)
concurrency: ConcurrencyConfig = field(default_factory=ConcurrencyConfig)
```

### 2.5. Temperature Optimizer Conditional

**File**: `lerobot/src/lerobot/scripts/rl/learner.py`

Made temperature optimizer and training conditional (SAC-only):

```python
# In make_optimizers_and_scheduler()
# SAC uses learned temperature, DrQ-v2 uses scheduled noise
if hasattr(policy, "log_alpha"):
    optimizer_temperature = torch.optim.Adam(params=[policy.log_alpha], ...)
    optimizers["temperature"] = optimizer_temperature

# In training loop
if "temperature" in optimizers:
    # SAC temperature optimization
    ...
```

### 3. PolicyFeature Shape Parsing

**File**: `lerobot/src/lerobot/policies/drqv2/modeling_drqv2.py`

Fixed `_setup_input_shapes()` to handle both dict (JSON) and `PolicyFeature` (parsed config) formats:

```python
# Low-dim state shape
if "observation.state" in config.input_features:
    state_feature = config.input_features.get("observation.state")
    if hasattr(state_feature, "shape"):
        self.low_dim_size = state_feature.shape[0]
    elif isinstance(state_feature, dict):
        self.low_dim_size = state_feature.get("shape", [0])[0]
    else:
        self.low_dim_size = 0
```

### 4. Pretrained Weight Loading

**File**: `lerobot/src/lerobot/scripts/rl/learner.py`

Added support for loading Genesis DrQ-v2 pretrained weights at learner startup:

```python
# Load pretrained weights if specified
pretrained_path = getattr(cfg.policy, "pretrained_path", None)
if pretrained_path is None:
    # Config parser may not preserve pretrained_path, read from saved JSON
    config_json_path = os.path.join(cfg.output_dir, "train_config.json")
    if os.path.exists(config_json_path):
        with open(config_json_path) as f:
            saved_cfg = json.load(f)
        pretrained_path = saved_cfg.get("policy", {}).get("pretrained_path")

if pretrained_path:
    checkpoint = torch.load(pretrained_path, map_location=device, weights_only=False)
    if "agent" in checkpoint:
        # RoboBase/Genesis format
        policy._load_robobase_weights(checkpoint["agent"])
```

### 5. RoboBase Weight Loading with Shape Mismatch Handling

**File**: `lerobot/src/lerobot/policies/drqv2/modeling_drqv2.py`

Modified `_load_robobase_weights()` to skip weights with shape mismatches instead of crashing:

```python
def _load_robobase_weights(self, state_dict):
    new_state_dict = {}
    skipped = []
    model_state = self.state_dict()

    for key, value in state_dict.items():
        new_key = self._map_robobase_key(key)
        if new_key is not None and new_key in model_state:
            if model_state[new_key].shape == value.shape:
                new_state_dict[new_key] = value
            else:
                skipped.append(f"{new_key}: {value.shape} vs {model_state[new_key].shape}")

    self.load_state_dict(new_state_dict, strict=False)
```

### 6. Frame Stacking in Actor

**File**: `lerobot/src/lerobot/scripts/rl/actor.py`

Added `FrameStackBuffer` class to stack observations across timesteps when `frame_stack > 1`:

```python
class FrameStackBuffer:
    """Buffer for stacking frames across timesteps."""

    def __init__(self, frame_stack: int, image_keys: list[str], state_key: str):
        self.frame_stack = frame_stack
        self.image_buffers = {key: deque(maxlen=frame_stack) for key in image_keys}
        self.state_buffer = deque(maxlen=frame_stack)

    def reset(self, obs: dict) -> dict:
        """Reset buffer with initial observation, filling with copies."""
        ...

    def update(self, obs: dict) -> dict:
        """Add new observation to buffer and return stacked observation."""
        ...
```

Usage in `act_with_policy()`:
- Initialize buffer after policy creation
- Stack initial observation after `env.reset()`
- Stack next observation after each `env.step()`
- Reset buffer on episode boundaries

### 7. HIL-SERL Config File

**File**: `outputs/hilserl_drqv2/train_config.json`

Created DrQ-v2 specific configuration matching Genesis checkpoint:

| Parameter | Value | Notes |
|-----------|-------|-------|
| `policy.type` | `drqv2` | Select DrQ-v2 policy |
| `frame_stack` | 3 | Match Genesis training |
| `input_features.observation.images.shape` | `[9, 84, 84]` | 3 frames × 3 RGB channels |
| `input_features.observation.state.shape` | `[54]` | 18 × 3 frames |
| `env.features.*.shape` | Single frame | Environment produces unstacked |
| `pretrained_path` | Genesis checkpoint | `/home/.../best_snapshot.pt` |
| `encoder_channels` | 32 | CNN feature channels |
| `bottleneck_size` | 50 | MLP bottleneck dim |
| `stddev_schedule` | `linear(1.0,0.1,50000)` | Exploration noise decay |
| `use_augmentation` | true | Random shift regularization |

## DrQ-v2 vs SAC Comparison

| Feature | SAC | DrQ-v2 |
|---------|-----|--------|
| Encoder | Pretrained ResNet | Custom CNN (trained) |
| Exploration | Entropy regularization | Scheduled noise |
| Regularization | None | Random shift augmentation |
| Temperature | Learned α | None |
| Normalization | ImageNet mean/std | x/255 - 0.5 |

## Policy Architecture

```
DrQV2Policy (~10.3M params)
├── Encoder (28.6K params)
│   └── 4-layer CNN per camera
│       └── 1 downsample conv (stride=2)
│       └── 3 post-downsample convs (stride=1)
├── ViewFusion
│   └── Flatten: (V, features) → (V * features,)
├── Actor (2.0M params)
│   └── MLPWithBottleneck
│       └── Bottleneck: visual/state → 50-dim
│       └── MLP: [256, 256] → action_dim
└── Critic (4.1M params, x2)
    └── MLPWithBottleneck (same structure)
```

## Usage

```bash
# Start learner (terminal 1)
uv run python -m lerobot.scripts.rl.learner \
    --config_path outputs/hilserl_drqv2/train_config.json

# Start actor (terminal 2)
uv run python -m lerobot.scripts.rl.actor \
    --config_path outputs/hilserl_drqv2/train_config.json
```

## Files Modified

- `lerobot/src/lerobot/scripts/rl/learner.py` - Encoder optimizer, temperature conditional, pretrained weight loading
- `lerobot/src/lerobot/scripts/rl/actor.py` - FrameStackBuffer for frame stacking
- `lerobot/src/lerobot/policies/drqv2/configuration_drqv2.py` - HIL-SERL config fields
- `lerobot/src/lerobot/policies/drqv2/modeling_drqv2.py` - PolicyFeature parsing, RoboBase weight loading with shape mismatch handling
- `lerobot/src/lerobot/configs/train.py` - Updated log_freq=100, save_freq=500

## Files Added

- `outputs/hilserl_drqv2/train_config.json` - DrQ-v2 HIL-SERL config (frame_stack=3)

## Key Design Decisions

1. **Frame stacking in actor, not environment**: The environment produces single frames, and the actor's `FrameStackBuffer` stacks them before passing to the policy. This keeps the environment simple and allows flexible frame stacking configuration.

2. **Pretrained path fallback to JSON**: The config parser doesn't always preserve `pretrained_path`, so the learner reads it directly from the saved JSON config as a fallback.

3. **Shape mismatch handling**: Instead of crashing on weight shape mismatches, `_load_robobase_weights` skips incompatible weights and logs them. This allows partial weight transfer when architecture differs.

## Related

- Devlog 037: Genesis DrQ-v2 inference (sim2real transfer)
- Devlog 038: IK grasp demo fixes
