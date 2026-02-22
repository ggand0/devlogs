# ACT Baseline Training — Architecture & Implementation Notes

## Overview

Training ACT (Action Chunking with Transformers) as a baseline for comparison against SmolVLA fine-tuning on the SO-101 10cm pick-and-place dataset (75 episodes, 22,458 frames).

Paper: [Learning Fine-Grained Bimanual Manipulation with Low-Cost Hardware](https://huggingface.co/papers/2304.13705)

## Architecture (lerobot defaults)

**Total parameters: ~52M** (not ~10M as commonly cited for the original paper)

### Parameter Breakdown

| Component | Params | Notes |
|---|---|---|
| 2× ResNet18 backbone | ~22.4M | ImageNet-pretrained, FrozenBatchNorm2d, one per camera |
| VAE encoder (4-layer transformer) | ~17.3M | Only used during training (CVAE objective) |
| Main encoder (4-layer transformer) | ~17.3M | Processes [latent, state, image_tokens] |
| Decoder (1-layer transformer) | ~5.4M | Cross-attends to encoder output |
| Projections & embeddings | ~0.4M | Linear projections, positional embeddings, action head |

### Why 52M vs. the "10M" from the paper

The original ACT paper used a single camera on the ALOHA bimanual setup with smaller transformer dimensions. lerobot's defaults are larger:

1. **2 cameras** (front + wrist) → 2× ResNet18 = 22.4M vs. 1× = 11.2M
2. **dim_model=512, dim_feedforward=3200** — larger than the paper's likely 256/1024
3. **4 VAE encoder layers** — adds 17.3M that only matter during training
4. **4 main encoder layers** at full width

### Model Architecture Diagram

From lerobot source (`modeling_act.py`):

```
                             Transformer
                             Used alone for inference
                             (acts as VAE decoder
                              during training)
                            ┌───────────────────────┐
                            │             Outputs    │
                            │                ▲       │
                            │     ┌─────►┌───────┐   │
               ┌──────┐     │     │      │Transf.│   │
               │      │     │     ├─────►│decoder│   │
          ┌────┴────┐ │     │     │      │(1 lyr)│   │
          │         │ │     │ ┌───┴───┬─►│       │   │
          │ VAE     │ │     │ │       │  └───────┘   │
          │ encoder │ │     │ │Transf.│              │
          │(4 lyrs) │ │     │ │encoder│              │
          └───▲─────┘ │     │ │(4lyrs)│              │
              │       │     │ └▲──▲─▲─┘              │
              │       │     │  │  │ │                │
            inputs    └─────┼──┘  │ image emb.       │
                            │   state emb.           │
                            └───────────────────────┘
```

### Encoder Input Tokens

The transformer encoder receives a sequence of tokens:
1. **Latent token** (1): Projected from VAE latent (32-dim → 512-dim), or zeros at inference
2. **Robot state token** (1): Projected from proprioceptive state (6-dim → 512-dim)
3. **Image feature tokens** (2 × H'×W'): ResNet18 feature maps flattened per camera

For 640×480 input images, ResNet18 layer4 outputs 20×15 = 300 spatial positions per camera. Total encoder sequence length: 1 + 1 + 300 + 300 = **602 tokens**.

This is where the memory goes — self-attention on 602 tokens at batch_size=64 with 4 layers is expensive.

### Decoder

- **1 layer** (not 7 — there's a [known bug](https://github.com/tonyzhaozh/act/issues/25) in the original ACT where only the first decoder layer is used; lerobot matches this)
- Input: `chunk_size` learnable query embeddings (DETR-style object queries)
- Cross-attends to encoder output
- Action head: Linear(512 → action_dim)

### VAE (Conditional VAE)

- Separate 4-layer transformer encoder
- Input tokens: [CLS, robot_state, action_1, ..., action_chunk_size]
- CLS token → Linear → (μ, log σ²) of latent distribution (32-dim)
- Training loss: L1 reconstruction + KL divergence (kl_weight=10.0)
- At inference: latent = zeros (no sampling)

### Vision Backbone

- ResNet18 with ImageNet-pretrained weights (`ResNet18_Weights.IMAGENET1K_V1`)
- FrozenBatchNorm2d (batch norm frozen, not trainable)
- Final layer4 feature map projected via 1×1 conv to dim_model=512
- 2D sinusoidal positional embeddings added to spatial features
- Separate backbone lr supported (`optimizer_lr_backbone`) but currently same as main lr

## Issue: No Image Resize in ACT

Unlike SmolVLA (`resize_with_pad` to 512×512 preserving aspect ratio) and pi0/pi0fast (224×224), **ACT has no built-in image resize**. It feeds raw dataset resolution directly into ResNet18.

With our 640×480 dataset images:
- ResNet18 layer4 outputs 20×15 = **300 spatial tokens per camera**
- 2 cameras → 602 total encoder tokens
- At 224×224 (ResNet18's intended input): 7×7 = 49 tokens per camera → **100 total**

This caused three compounding problems:
1. **OOM at batch_size=64** — 602-token attention maps blew past 24GB VRAM
2. **Slow training** — data loading 640×480 video frames via pyav: ~0.43s/step (39% of total step time)
3. **Wasted compute** — ResNet18 was designed for 224×224; 6× more spatial tokens than needed with no benefit

**Fix for next run**: Add resize to 224×224 (or 240×320 for 4:3 aspect ratio) before feeding into ACT. This would cut encoder tokens by 6×, fix the OOM at batch_size=64, and significantly reduce data loading time.

## Training Config

```
policy.type              = act
policy.chunk_size        = 50        # action prediction horizon
policy.n_action_steps    = 50        # all predicted actions used
policy.use_vae           = true      # CVAE training objective
policy.dim_model         = 512
policy.n_heads           = 8
policy.dim_feedforward   = 3200
policy.n_encoder_layers  = 4
policy.n_decoder_layers  = 1
policy.n_vae_encoder_layers = 4
policy.latent_dim        = 32
policy.kl_weight         = 10.0
policy.dropout           = 0.1
policy.vision_backbone   = resnet18

batch_size               = 32        # reduced from 64 (OOM due to 480p input)
steps                    = 50000     # stopping at 40k (matches SmolVLA sample count)
save_freq                = 10000     # checkpoints at 10k/20k/30k/40k/50k
optimizer                = AdamW
optimizer.lr             = 1e-5
optimizer.weight_decay   = 1e-4
scheduler                = none
dataset                  = gtgando/so101_pick_place_10cm_v1
```

## Training Results

- **Loss at 37k**: 0.046 (L1 + KL combined)
- **Step time**: ~1.1s/step (updt_s: 0.47, data_s: 0.43)
- **Gradient norm**: ~3.6
- **Checkpoints saved**: 10k, 20k, 30k (197MB each)

## Comparison: ACT vs SmolVLA

| | ACT | SmolVLA |
|---|---|---|
| Total params | 52M | 500M+ |
| Trainable params | 52M (all) | ~50M (action expert + projections) |
| Vision | 2× ResNet18 (ImageNet) | SmolVLM (frozen) |
| Language | None | SmolLM2 (frozen) |
| Training | From scratch | Fine-tune pretrained VLA |
| Image input | 640×480 raw (no resize) | 512×512 (resize_with_pad) |
| Batch size | 32 | 64 |
| Steps | 40k (stopped early) | 20,000 |
| Samples seen | 1.28M | 1.28M |
| Step time | ~1.1s | ~1.7s |
| Samples/sec | 29.1 | 37.0 |
| Chunk size | 50 | 50 |
| Final loss | 0.046 (L1+KL) | 0.006 (L1) |
| Loss | L1 + KL | L1 |

## Memory Issue

First attempt at batch_size=64 OOM'd on 24GB RTX 3090:
- ACT process alone used 20.94 GiB
- Tried to allocate 76 MiB more with only 392 MiB free
- Root cause: 602-token encoder sequence (from 640×480 images) × batch_size=64 × 4 layers of self-attention
- Fix: batch_size=32 + `PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True`
- Proper fix: resize images to 224×224 before feeding into ResNet18

## Files

- Training script: `scripts/train_act.sh`
- Output dir: `outputs/train/act_so101_10cm`
- Log: `act_train.log`
- lerobot ACT config: `lerobot/src/lerobot/policies/act/configuration_act.py`
- lerobot ACT model: `lerobot/src/lerobot/policies/act/modeling_act.py`
