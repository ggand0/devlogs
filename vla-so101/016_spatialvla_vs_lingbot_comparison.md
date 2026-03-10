# SpatialVLA vs LingBot-VLA Comparison

Evaluated two depth-aware VLA foundation models for fine-tuning on the SO-101 pick-and-place task.

## Overview

| Aspect | SpatialVLA | LingBot-VLA |
|---|---|---|
| Paper | [arXiv:2501.15830](https://arxiv.org/abs/2501.15830) | [arXiv:2601.18692](https://arxiv.org/abs/2601.18692) (Jan 2026) |
| Code | [SpatialVLA/SpatialVLA](https://github.com/SpatialVLA/SpatialVLA) | [Robbyant/lingbot-vla](https://github.com/Robbyant/lingbot-vla) |
| Size | 4B (PaliGemma2 backbone) | ~4B (Qwen2.5-VL-3B + action expert) |
| Pretraining | 1.1M episodes (Open X-Embodiment + RH20T) | 20,000 hours real-world, 9 dual-arm platforms |
| Training framework | HuggingFace Transformers | VeOmni + FSDP2 (custom) |
| LoRA support | Yes (via PEFT) | No — full fine-tuning only |
| Min training hardware | Single GPU (LoRA on 3090) | 8+ GPUs (FSDP2 required) |
| Data format | RLDS (TFRecord) | Custom LeRobot fork |
| Target robots | General (single-arm, dual-arm) | Primarily dual-arm (7-DOF per arm) |

## Depth Integration

The two models take fundamentally different approaches to incorporating depth:

**SpatialVLA — Ego3D Position Encoding:**
- Runs ZoeDepth on RGB at runtime to estimate depth (no sensor needed)
- Back-projects pixels to 3D point cloud using camera intrinsics
- Encodes 3D positions with sinusoidal functions → MLP → added to SigLIP vision tokens
- Depth modifies WHERE tokens are in 3D space (positional information)
- Frozen SigLIP encoder — depth doesn't change the visual features, only their spatial encoding

**LingBot-VLA — Depth Feature Distillation:**
- Uses a dedicated LingBot-Depth model (ViT-L/14, trained with Masked Depth Modeling on RGB-D data)
- Learnable query tokens attend to depth features via cross-attention
- Depth knowledge is distilled into the VLA through a projection layer + contrastive loss
- Requires actual RGB-D sensor data (or depth completion model)
- Depth is a separate knowledge stream, not a position modification

SpatialVLA's approach is more elegant for deployment (no depth sensor needed), while LingBot's approach arguably captures richer depth semantics but adds hardware requirements.

## Action Prediction

**SpatialVLA — Adaptive Action Grid Tokenization:**
- Outputs 3 autoregressive tokens → decoded via learned discretization grids into 7D continuous action
- Each token is generated sequentially by the LM head
- Potential issue: action chunk boundary jitter (the shaking we observed with SmolVLA's similar approach)

**LingBot-VLA — Flow Matching:**
- Predicts entire 50-step action chunks in one forward pass (not autoregressive)
- Continuous action output via learned velocity fields
- Smoother trajectories — no discretization artifacts or boundary jitter
- One forward pass covers 1 second of robot actions at 50Hz

Flow Matching is a significant architectural advantage for manipulation smoothness.

## Performance

**SpatialVLA:**
- Strong on SimplerEnv simulation benchmarks
- Designed for general single-arm and multi-arm setups
- No published GM-100 real-world benchmark results

**LingBot-VLA (GM-100 real-world benchmark, 100 tasks, 3 platforms):**

| Model | Avg Success Rate | Avg Progress Score |
|---|---|---|
| WALL-OSS | ~7.6% | ~16.0% |
| GR00T N1.6 | ~7.6% | ~16.0% |
| pi0.5 | 13.0% | 27.7% |
| LingBot-VLA (no depth) | >13% | >27.7% |
| **LingBot-VLA (with depth)** | **17.3%** | **35.4%** |

LingBot-VLA with depth is SOTA on GM-100, beating pi0.5 by ~4.3pp. Depth integration provides significant boost on geometry-sensitive tasks (insertion, stacking, folding).

**LingBot-VLA (RoboTwin 2.0 simulation, 50 tasks):**
- Clean scenes: 88.6% success
- Randomized scenes (lighting, clutter): 86.7% success

## Feasibility for SO-101 Setup

### SpatialVLA — Feasible

- LoRA fine-tuning fits on RTX 3090 (~18-22GB with batch_size=1, gradient checkpointing, bf16, rank 32)
- Standard HuggingFace ecosystem — familiar tooling
- RLDS format with existing conversion tools (Ke-Wang1017/lerobot_rlds)
- No depth sensor required for training (ZoeDepth at runtime)
- Can optionally swap in real D405 depth at inference to save VRAM (~1-2GB from dropping ZoeDepth)
- Single-arm 6-DOF action space is natively supported

### LingBot-VLA — Not Feasible Currently

- No LoRA support → can't fine-tune on single 3090
- FSDP2 multi-GPU training required (8+ GPUs implied by docs)
- Designed for dual-arm robots (action dim 14-16), SO-101 is single-arm 6-DOF
- Custom VeOmni framework, not standard HF ecosystem
- Bleeding-edge dependencies (PyTorch 2.8.0, CUDA 12.8, Flash Attention 2.8.3)
- RGB-D sensor required for depth variant (we have this now, but training infra is the blocker)

## Decision

**Start with SpatialVLA.** It fits our hardware constraints (single 3090, single-arm SO-101, D405 camera) and has a clear fine-tuning path. The depth recording infrastructure on `feat/depth-recording` gives us the option to use real D405 depth at inference time.

**Revisit LingBot-VLA if:**
- They release LoRA support or single-GPU training configs
- We get access to multi-GPU training (cloud/lab)
- They add single-arm robot configurations
- The Flow Matching action prediction is worth porting as a standalone idea

## Interesting Ideas from LingBot to Watch

- **Flow Matching for action prediction**: produces smoother trajectories than autoregressive tokenization. Could potentially be adapted independently of the full LingBot architecture.
- **Masked Depth Modeling**: self-supervised depth pretraining that treats sensor noise/missing regions as natural masks. Relevant if we build a depth fusion pipeline.
- **Mixture-of-Transformers**: separate action expert with shared attention. Clean separation of visual understanding and motor control.
