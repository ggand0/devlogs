# 018 — SpatialVLA LoRA Training Results (Run 1)

## Overview

First LoRA fine-tuning run of SpatialVLA-4B on SO-101 pick-and-place dataset completed successfully.

- **Duration**: 11h 34m on single RTX 3090
- **Steps**: 9,814 (7 epochs × 1,402 steps/epoch)
- **Final loss**: 0.0799 (avg: 0.3355)

## Dataset

- **Source**: `gtgando/so101_pick_place_10cm_v3` (LeRobot v2 format)
- **Episodes**: 75
- **Transitions**: 22,436 (~300 steps/episode at 30fps)
- **Camera**: Wrist-mounted Intel RealSense D405 (480×640 RGB)
- **Actions**: 6D absolute joint positions (5 joints + 1 gripper) in degrees
- **Converted to RLDS**: `~/ggando/ml/rlds_datasets/so101_pick_place/1.0.0/` (1.3GB, 16 shards)

## Training Configuration

| Parameter | Value |
|---|---|
| Base model | `IPEC-COMMUNITY/spatialvla-4b-224-pt` |
| LoRA rank | 32 |
| LoRA alpha | 32 |
| LoRA target | `linear` (all linear layers) |
| Trainable params | ~59M / 4B (1.45%) |
| Batch size | 2 × 8 grad accum = 16 effective |
| Learning rate | 5e-4 (linear decay) |
| Warmup | 0.5% of steps (~49 steps) |
| Epochs | 7 |
| Action chunking | 3 (predict current + 3 future) |
| Precision | bf16 + tf32 |
| DeepSpeed | ZeRO-1 |
| flash-attn | Disabled (CUDA version mismatch) |

**Output**: `~/ggando/ml/SpatialVLA/outputs/so101_lora_r32_bs16_ep7_2026-03-19_03-01/`

## Loss Curve

```
Step      Loss     LR
──────────────────────────
   50    1.5709   5.00e-4
  100    0.9015   4.97e-4
  200    0.8112   4.92e-4
 1000    0.5913   4.51e-4
 2000    0.4594   4.00e-4
 3000    0.3942   3.49e-4
 4000    0.3611   2.98e-4
 5000    0.3751   2.47e-4
 6000    0.2631   1.95e-4
 7000    0.1856   1.44e-4
 8000    0.1394   9.31e-5
 9000    0.1141   4.19e-5
 9500    0.0914   1.63e-5
 9800    0.0799   9.73e-7
```

Loss dropped smoothly from ~1.57 → 0.08. No signs of divergence. The bump at step 5000 (0.3751) is likely an epoch boundary effect. Loss was still decreasing at end of training — more epochs may help, but risk of overfitting on 75 episodes.

## Checkpoints

| Checkpoint | Step | Size |
|---|---|---|
| checkpoint-6000 | 6000 | LoRA adapter |
| checkpoint-8000 | 8000 | LoRA adapter |
| checkpoint-9814 | 9814 (final) | 113MB adapter |

Each checkpoint contains `adapter_model.safetensors` (LoRA weights) + `adapter_config.json` + tokenizer/processor files.

## Dataset Statistics (from `ds_stats.json`)

Action normalization bounds (BOUNDS_Q99), used for denormalization at inference:

| Joint | Q01 | Q99 | Mean | Std |
|---|---|---|---|---|
| shoulder_pan | -44.97° | 11.56° | -12.18° | 16.30° |
| shoulder_lift | -104.26° | 31.91° | -43.17° | 49.18° |
| elbow_flex | -42.95° | 98.15° | 38.29° | 48.11° |
| wrist_flex | 30.33° | 71.56° | 52.38° | 9.07° |
| wrist_roll | 63.08° | 130.95° | 90.02° | 12.19° |
| padding (dim 5) | 0.0 | 0.0 | 0.0 | 0.0 |
| gripper | 0.90° | 46.87° | 12.63° | 14.98° |

Dimension 5 is always zero (padding to meet SpatialVLA's 7D requirement). `action_normalization_mask[5] = False` prevents division by zero during normalization.

## Inference Pipeline

Implemented in `scripts/infer_spatialvla.py` + `scripts/infer_spatialvla.sh`.

1. Load base model from `~/ggando/ml/pretrained/spatialvla-4b-224-pt`
2. Attach LoRA adapter via `peft.PeftModel.from_pretrained` then `merge_and_unload()` for faster inference
3. Inject `ds_stats.json` into `processor.statistics["so101_pick_place/1.0.0"]` (also baked into checkpoint's `processor_config.json`)
4. Open SO-101 follower with wrist RealSense only (overhead skipped — not in training mixture)
5. Reset to home pose
6. Loop:
   - Capture wrist image → `processor(images=[img], text=prompt)` → `model.predict_action(...)`
   - `processor.decode_actions(gen, unnorm_key="so101_pick_place/1.0.0")` returns `(action_chunk_size=4, 7)`
   - For each of the 4 actions: drop dim 5 (padding), clamp 5 joints, send via `bus.sync_write("Goal_Position", ...)`

### Dependencies installed in SpatialVLA venv

The SpatialVLA venv was missing several lerobot deps. Installed via uv:

```
draccus pyserial deepdiff feetech-servo-sdk pyrealsense2
```

## First Inference Test (2026-04-22)

Pipeline runs end-to-end on hardware:

```
Policy load:        ~15s (base + LoRA merge + cuda())
RealSense connect:  <1s
Inference:          ~510ms per chunk (4 actions × 7 dims)
                    → effective control rate ~8 Hz
                    (training was 30 Hz)
```

**Behavior**: policy runs but produces unusable motion. Not yet diagnosed
which of the following is the dominant cause:

- **Control-rate mismatch**: training at 30 Hz, executing at 8 Hz. Each absolute joint command is held 4× longer than during recording.
- **Overfitting**: 75 episodes is small; 7 epochs may have memorized scene specifics (lighting, cube position) that don't match the live setup.
- **Prompt drift**: dataset task is exactly `"Pick up the cube and place it in the bowl"` (no color); inference tested with `"Pick up the red cube and place it in the bowl"`.
- **Action representation**: SpatialVLA's translation tokenizer treats action dims 0-2 as cartesian xyz and applies a spherical transform before binning. Our actions are joint angles in degrees, normalized to [-1, 1]. Encode/decode is round-trip consistent so this should work in principle, but quantization through spherical bins may degrade joint-space accuracy more than xyz-space accuracy.
- **Domain gap**: 224×224 resize + lighting/camera differences between training and live capture.

### Next debugging steps

1. Re-run with the exact training prompt (no "red")
2. Try `CHECKPOINT=checkpoint-6000` and `checkpoint-8000` (less overfit)
3. Drop `EXEC_PER_CHUNK=1` so each action gets a fresh observation
4. Add a `--debug-dump` flag that saves the wrist image + predicted 7D action to disk per step for offline inspection
5. Compare predicted joint values against home pose / dataset means — sanity check whether the model is just predicting the mean trajectory

## Scripts

- **RLDS conversion**: `scripts/convert_to_rlds.py`
- **Training**: `scripts/train_spatialvla_lora.sh`
- **Inference**: `scripts/infer_spatialvla.py` + `scripts/infer_spatialvla.sh`
- **Setup details**: `devlogs/017_spatialvla_finetuning_setup.md`
