# SmolVLA v3 Dual-Camera Training

## Setup

- **Base model**: `lerobot/smolvla_base` (HuggingFace Hub)
- **Dataset**: `gtgando/so101_pick_place_10cm_v3` — 75 episodes, 22436 frames
- **Cameras**: wrist (RealSense D405) + overhead (Logitech C920), both 640x480 @ 30fps
- **Task**: Pick up the cube and place it in the bowl
- **Script**: `scripts/train_smolvla_v3_dual.sh`

### Training params

| Param | Value |
|---|---|
| batch_size | 64 |
| steps | 20000 |
| save_freq | 5000 |
| log_freq | 100 |
| num_workers | 4 |
| image_transforms | enabled |
| video_backend | pyav |
| wandb | disabled |

## Training progress

Training started 2026-02-21 ~17:18.

### Loss at all checkpoints

| Step | Loss | Grad norm | LR | Wall time (cumulative) |
|---|---|---|---|---|
| 5000 | 0.011 | 0.18 | ~8.5e-05 | ~2h 35m |
| 10000 | 0.010 | 0.19 | 7.6e-05 | ~5h 12m |
| 15000 | 0.007 | 0.13 | ~4.8e-05 | ~7h 48m (est) |
| 20000 | 0.005 | 0.11 | 2.7e-05 | ~10h 24m |

### Timing

- ~1.27s per update step, ~0.57s data loading
- ~1.6-1.9s/step total (including data)
- First half (0-10k): ~5h 12m
- Second half (10k-20k, resumed): ~5h 12m
- **Total wall time: ~10h 24m**
- 56.77 epochs over the 75-episode dataset

### Resume

Training was paused at 10k and resumed with:

```bash
./scripts/train_smolvla_v3_dual.sh --resume-from outputs/train/smolvla_so101_10cm_v3_dual/checkpoints/010000
```

This uses a custom `--resume-from` flag in `train.py` that translates to lerobot's native `--resume=true --config_path=...`, restoring optimizer/scheduler/step state.

## Inference

```bash
uv run python scripts/infer.py --record --direct-reset --num-rollouts 5
```

Uses `outputs/train/smolvla_so101_10cm_v3_dual/checkpoints/last/pretrained_model` by default. `--record` saves PiP video with audio to `recordings/infer_{timestamp}.mp4`.

### Eval results (20k checkpoint)

- **Typical success rate: 60-80%** across multiple 5-episode runs
- Task: pick up cube and place in bowl
- Shaking observed during execution — likely from action chunk boundary jitter (common with ACT-style policies)
- Possible improvements: more training data (75 eps is low), more steps (loss still dropping at 20k), temporal ensembling
- **Demo video**: `recordings/infer_20260222_122103_100%_funny_trimmed.mp4` — [posted on X](https://x.com/gtgando/status/2025427031102722300)

## Notes

- First training run with both cameras. Previous v2 runs excluded overhead due to black frames from ep 71+.
- v3 dataset recorded on `fix/teleop-lag` branch with action-first loop reorder. 5 bad episodes removed (3, 17, 18, 26, 45).
- Loss converged to 0.005 by step 20k. Gradient norm dropped from 0.18 to 0.11, stable convergence.
