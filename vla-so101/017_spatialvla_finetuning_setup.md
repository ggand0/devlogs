# 017 — SpatialVLA LoRA Fine-tuning Setup

## Goal

Fine-tune SpatialVLA-4B on our SO-101 pick-and-place dataset (75 episodes, wrist camera) using LoRA on a single RTX 3090.

## Pipeline Overview

Three-step pipeline: LeRobot v2 → RLDS → SpatialVLA LoRA training.

```
LeRobot v2 dataset          RLDS TFRecords              SpatialVLA LoRA
(parquet + AV1 MP4)    →    (TFDS format)          →    (PyTorch training)
~400MB                      1.3GB / 16 shards           ~7.6GB model
```

## Step 1: RLDS Conversion

**Script**: `scripts/convert_to_rlds.py`

Reads LeRobot v2 format directly (parquet for action/state, AV1 MP4 for images) and outputs RLDS TFRecords via `tensorflow_datasets.GeneratorBasedBuilder`.

- Uses PyAV for AV1 decoding (OpenCV headless can't decode AV1)
- Stores 6D actions/state as float32, images as JPEG-encoded uint8
- Output: `~/ggando/ml/rlds_datasets/so101_pick_place/1.0.0/` (75 episodes, 298-300 steps each)

```bash
cd ~/ggando/ml/SpatialVLA && source .venv/bin/activate
python ~/ggando/ml/vla-so101/scripts/convert_to_rlds.py \
    --lerobot-dir ~/.cache/huggingface/lerobot/gtgando/so101_pick_place_10cm_v3 \
    --output-dir ~/ggando/ml/rlds_datasets
```

## Step 2: SpatialVLA Dataset Registration

Three files modified in `~/ggando/ml/SpatialVLA/`:

### `data/oxe/configs.py`
```python
"so101_pick_place/1.0.0": {
    "image_obs_keys": {"primary": "wrist_image", "secondary": None, "wrist": None},
    "depth_obs_keys": {"primary": None, "secondary": None, "wrist": None},
    "state_obs_keys": ["state"],
    "state_encoding": StateEncoding.JOINT,
    "action_encoding": ActionEncoding.EEF_POS,
    "aux_kwargs": {
        "absolute_action_mask": [True] * 7,
        "action_normalization_mask": [True] * 5 + [False] + [True],
    },
},
```

### `data/oxe/transforms.py`
```python
def so101_pick_place_dataset_transform(trajectory):
    action = trajectory["action"]  # (T, 6)
    trajectory["action"] = tf.concat(
        [action[:, :5], tf.zeros_like(action[:, :1]), action[:, 5:]],
        axis=-1,
    )  # (T, 7)
    return trajectory
```

### `data/oxe/mixtures.py`
```python
"so101_pick_place": [("so101_pick_place/1.0.0", 1.0)],
```

### Key Design Decisions

**Why register as EEF_POS instead of JOINT_POS?**

SpatialVLA's `make_oxe_dataset_kwargs()` in `__init__.py` has a hard validation that only allows `EEF_POS` and `EEF_R6` action encodings. Rather than patching core code, we register as `EEF_POS` and override the masks via `aux_kwargs` (which get applied AFTER the validation).

**Why pad 6D → 7D?**

SpatialVLA's `SpatialActionTokenizer` hardcodes 7D input, split as [translation(3), rotation(3), gripper(1)]. Our 5 joints + 1 gripper get padded: `[j1, j2, j3, j4, j5, 0, gripper]`. The padding dim (index 5) has `action_normalization_mask=False` to avoid division by zero during BOUNDS_Q99 normalization.

**Why `absolute_action_mask = [True]*7`?**

Our dataset stores absolute joint positions (not deltas). Setting all dims to absolute means padding beyond trajectory end repeats the last valid action (correct for absolute positions) rather than zeroing out (which would only be correct for delta actions).

**Camera mapping**: Wrist camera mapped to `"primary"` because `load_camera_views=("primary",)` is hardcoded in `dataset.py`. Proprioception (`load_proprio=False`) is not loaded during training.

## Step 3: LoRA Training

**Script**: `scripts/train_spatialvla_lora.sh`

**Model**: `IPEC-COMMUNITY/spatialvla-4b-224-pt` (7.6GB, downloaded to `~/ggando/ml/pretrained/spatialvla-4b-224-pt/`)

| Parameter | Value | Notes |
|---|---|---|
| LoRA rank | 32 | Standard for 4B models |
| LoRA alpha | 32 | = rank |
| LoRA target | `linear` | All linear layers + projector + ego3d heads |
| Batch size | 2 | RTX 3090 24GB with gradient checkpointing |
| Grad accumulation | 8 | Effective batch = 16 |
| Learning rate | 5e-4 | Standard LoRA LR |
| Epochs | 50 | ~70k steps total |
| Action chunking | 3 | Predict current + 3 future actions |
| flash-attn | Disabled | System CUDA 13.1 vs PyTorch CUDA 12.6 mismatch |
| DeepSpeed | ZeRO-1 | No memory benefit on 1 GPU but compatible |

```bash
cd ~/ggando/ml/SpatialVLA && source .venv/bin/activate
bash ~/ggando/ml/vla-so101/scripts/train_spatialvla_lora.sh
```

## Environment

SpatialVLA uses a dedicated uv venv at `~/ggando/ml/SpatialVLA/.venv/` (Python 3.10):
- PyTorch 2.10.0+cu126
- TensorFlow 2.15.0 (for RLDS loading)
- dlimp (custom fork from SpatialVLA)
- flash-attn NOT installed (CUDA version mismatch)

## Inference (TODO)

At inference time:
1. Load LoRA adapter from checkpoint
2. Model outputs 7D normalized action → denormalize using saved `ds_stats.json`
3. Discard dimension 5 (padding) → 6D absolute joint positions
4. Send to SO-101 follower arm

Dataset statistics for denormalization are saved automatically during training at `{output_dir}/ds_stats.json`.
