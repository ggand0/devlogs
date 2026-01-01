# Devlog 037: Hydra to Argparse+YAML Config Refactoring

## Summary

Refactored `train_image_rl.py` from Hydra-based config to simple argparse + YAML loading.
This removes the Hydra dependency while maintaining full RoboBase compatibility.

## Motivation

- Hydra's decorator-based approach (`@hydra.main()`) is opaque and hard to debug
- Config files were embedded in script directory structure
- Wanted simpler config system matching `train_lift.py` pattern in `./configs/`

## Files Created/Modified

### New Files

1. **`src/training/config_loader.py`** - Hydra replacement
   - `ROBOBASE_DEFAULTS` - Base training params extracted from robobase
   - `DRQV2_DEFAULTS` - DrQ-v2 method defaults with actor/critic/encoder configs
   - `load_config(path)` - Merges defaults with user YAML
   - `instantiate(cfg)` - Replaces `hydra.utils.instantiate()` for `_target_`/`_partial_` patterns

2. **`configs/drqv2_lift_s3.yaml`** - Stage 3 curriculum config

### Modified Files

1. **`src/training/train_image_rl.py`** - Now uses argparse
   - `--config` flag for YAML config file
   - `--resume` flag for snapshot resumption
   - Monkey-patches `hydra.utils.instantiate` for RoboBase compatibility

## Config Verification

Verified that all config values are actually used during training:

```
=== KEY CONFIG VALUES ===
num_train_envs: 8
batch_size: 256
frame_stack: 3
num_train_frames: 1000000

=== ENV CONFIG ===
curriculum_stage: 3
reward_version: v11
image_size: 84
episode_length: 200

=== METHOD CONFIG (NETWORK SIZES) ===
actor_model.mlp_nodes: [256, 256]   # Overrides default [1024, 1024]
critic_model.mlp_nodes: [256, 256]  # Overrides default [1024, 1024]
encoder_model.channels: 32
```

Verified agent network structure shows correct sizes:
```python
Actor:
  (main_mlp): Sequential(
    (0): Linear(in_features=100, out_features=256, bias=True)  # 256!
    (3): Linear(in_features=256, out_features=256, bias=True)  # 256!
  )
```

## How Config Merging Works

```python
# 1. Start with RoboBase defaults
cfg = OmegaConf.create(ROBOBASE_DEFAULTS)

# 2. Merge DrQ-v2 method defaults
cfg = OmegaConf.merge(cfg, OmegaConf.create(DRQV2_DEFAULTS))

# 3. Merge user YAML (overrides)
cfg = OmegaConf.merge(cfg, OmegaConf.create(user_cfg))
```

OmegaConf does deep merging, so nested overrides like:
```yaml
method:
  actor_model:
    mlp_nodes: [256, 256]  # Only this key overridden
```

...properly override just that key while keeping other defaults.

## Config Value Flow

```
configs/drqv2_lift_s3.yaml
    ↓
load_config() merges with defaults
    ↓
cfg passed to Workspace(cfg=cfg, ...)
    ↓
├── SO101Factory uses: cfg.env.*, cfg.num_train_envs, cfg.pixels, cfg.frame_stack
├── Agent instantiate uses: cfg.method.actor_model, cfg.method.critic_model, ...
└── Replay buffer uses: cfg.batch_size, cfg.replay.*
```

## Usage

```bash
# Train with default config
MUJOCO_GL=egl uv run python src/training/train_image_rl.py

# Train with specific config
MUJOCO_GL=egl uv run python src/training/train_image_rl.py --config configs/drqv2_lift_s3.yaml

# Resume training
MUJOCO_GL=egl uv run python src/training/train_image_rl.py \
    --config configs/drqv2_lift_s3.yaml \
    --resume runs/image_rl/20260101_120000/snapshots/latest_snapshot.pt
```

## Key Config Parameters

| Parameter | Default | Config Override | Used By |
|-----------|---------|-----------------|---------|
| `num_train_envs` | 1 | 8 | AsyncVectorEnv parallelization |
| `batch_size` | 256 | 256 | Replay buffer sampling |
| `frame_stack` | 1 | 3 | FrameStack wrapper |
| `mlp_nodes` | [1024,1024] | [256,256] | Actor/Critic networks |
| `curriculum_stage` | 3 | 3 | LiftCubeCartesianEnv reset |
| `reward_version` | v11 | v11 | Reward shaping function |

## Notes

- The `instantiate()` function handles `_target_` and `_partial_` fields recursively
- RoboBase's Workspace is used directly (not custom SO101Workspace)
- Config is copied to run directory for reproducibility
- Monkey-patching `hydra.utils.instantiate` ensures RoboBase internals work unchanged
