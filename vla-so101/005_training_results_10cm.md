# Training Results — 10cm Pick-and-Place Dataset

Dataset: `gtgando/so101_pick_place_10cm_v1` (75 episodes, 22,458 frames, 2 cameras @ 640×480)

## SmolVLA Fine-tuning

- **Steps**: 20,000 (batch_size=64) → 1.28M samples, ~57 epochs
- **Duration**: 9h 38m
- **Step time**: ~1.73s/step (updt_s: ~1.19, data_s: ~0.53)
- **Final loss**: 0.006 (L1)
- **Final lr**: 2.7e-05 (cosine decay from 1e-4)
- **Gradient norm**: ~0.12
- **Checkpoints**: 5k, 10k, 15k, 20k (865MB each)
- **Image preprocessing**: `resize_with_pad` to 512×512 (aspect ratio preserved, padded)

## ACT Baseline

- **Steps**: 50,000 target, stopping at 40k (batch_size=32) → 1.28M samples at 40k, ~57 epochs
- **Duration**: ~12h at 40k (estimated)
- **Step time**: ~1.1s/step (updt_s: ~0.47, data_s: ~0.43)
- **Loss at 37k**: 0.046 (L1 + KL, not directly comparable to SmolVLA's pure L1)
- **lr**: 1e-5 (constant, no scheduler)
- **Gradient norm**: ~3.6
- **Checkpoints**: 10k, 20k, 30k, 40k (197MB each)
- **Image preprocessing**: None — raw 640×480 fed directly into ResNet18

## Known Issue: ACT Image Resolution

ACT has no built-in image resize. Unlike SmolVLA (which does `resize_with_pad` to 512×512) and pi0/pi0fast (224×224), ACT feeds raw dataset resolution into ResNet18.

With 640×480 inputs, ResNet18 layer4 produces 20×15 = 300 spatial tokens per camera. With 2 cameras, the encoder processes 602 tokens total. This causes:
- **Excessive memory**: OOM at batch_size=64 on 24GB RTX 3090, had to reduce to 32
- **Slow training**: data_s dominates — decoding 640×480 video frames via pyav for 2 cameras takes ~0.43s/step
- **Unnecessary compute**: ResNet18 is designed for 224×224 inputs; 480p produces 6× more spatial tokens than needed

For future runs: add a resize to 224×224 (or 240×320 to preserve 4:3 aspect ratio) before feeding into ACT. This would reduce encoder tokens from 602 to ~102, cut memory by ~6×, and significantly speed up training.

## Eval Results

### SmolVLA 20k — 2025-02-17

- **Checkpoint**: `smolvla_so101_10cm/checkpoints/020000/pretrained_model`
- **Task**: Pick up the cube and place it in the bowl (10cm range)
- **Success rate**: 80% (4/5 episodes)
- **Episode duration**: 10s, 30fps
- **Video**: `recordings/infer_20260217_150625.mp4`

## Sample Efficiency Comparison

At matched sample counts (1.28M samples):
- SmolVLA: 20k steps × batch 64, loss 0.006
- ACT: 40k steps × batch 32, loss ~0.046

SmolVLA converges to a much lower L1 loss, though the losses aren't directly comparable (ACT includes KL divergence in its total loss). The L1 component alone (`l1_loss` in ACT) would need to be checked separately for a fair comparison.

## Throughput

| | SmolVLA | ACT |
|---|---|---|
| Step time | 1.73s | 1.1s |
| Update time | 1.19s | 0.47s |
| Data loading | 0.53s | 0.43s |
| Samples/sec | 37.0 | 29.1 |
| batch_size | 64 | 32 |
| GPU | RTX 3090 24GB | RTX 3090 24GB |

ACT's model forward+backward (0.47s) is 2.5× faster than SmolVLA's (1.19s), but the halved batch size means lower overall throughput. Data loading is the bottleneck for ACT (~39% of step time), largely due to decoding 640×480 video without downsampling.
