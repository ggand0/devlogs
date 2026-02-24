# ACT Baseline and SmolVLA Wrist-Only Training

Both trained on the v3 dataset (`gtgando/so101_pick_place_10cm_v3`, 75 episodes, 22436 frames).

![Training curves](training_curves_v3.png)

## ACT v3 Dual-Camera

From-scratch ACT training with aspect-ratio-preserving resize (`resize_imgs=[224,224]` via `_resize_with_pad`).

### Config

| Param | Value |
|---|---|
| Policy | ACT (52M params) |
| Backbone | 2x ResNet18 (ImageNet pretrained) |
| Cameras | wrist + overhead |
| Chunk size | 50 |
| n_action_steps | 50 |
| resize_imgs | [224, 224] |
| Optimizer | AdamW, lr=1e-5, wd=1e-4 |
| LR scheduler | none |
| Batch size | 64 |
| Steps | 20,000 |
| Save freq | 1,000 |

### Training

- Started: 2026-02-22 15:30, ended: 2026-02-23 02:00 (~10.5h)
- ~1.2s/step
- Final loss: 0.052, grad norm: 3.915

### Loss curve

| Step | Loss |
|---|---|
| 1k | 0.961 |
| 5k | 0.142 |
| 10k | 0.080 |
| 15k | 0.061 |
| 20k | 0.052 |

### Artifacts

- Checkpoints: `outputs/train/act_so101_10cm_v3_dual/checkpoints/{001000..020000,last}/`
- Log: `nohup_act_v3.out`

---

## SmolVLA v3 Wrist-Only

Same setup as the dual-cam SmolVLA run (devlog 011) but with `--exclude-cameras overhead`.

### Config

| Param | Value |
|---|---|
| Policy | SmolVLA (fine-tuned from `lerobot/smolvla_base`) |
| Cameras | wrist only |
| Optimizer | AdamW, lr=1e-4, wd=1e-10 |
| LR scheduler | CosineDecayWithWarmup (1k warmup, 30k decay) |
| Batch size | 64 |
| Steps | 20,000 |
| Save freq | 5,000 |

### Training

- Started: 2026-02-23 06:59, ended: 2026-02-23 17:08 (~10h)
- ~1.7s/step
- Final loss: 0.006, grad norm: 0.109, lr: 2.7e-05

### Loss curve

| Step | Loss |
|---|---|
| 1k | 0.032 |
| 5k | 0.017 |
| 10k | 0.011 |
| 15k | 0.007 |
| 20k | 0.006 |

### Artifacts

- Checkpoints: `outputs/train/smolvla_so101_10cm_v3_wrist/checkpoints/{005000,010000,015000,020000,last}/`
- Log: `nohup_smolvla_v3_wrist.out`

---

## Comparison

All three models on the v3 dataset (75 episodes, pick-and-place):

| Model | Cameras | Final Loss | Grad Norm | Train Time | Params |
|---|---|---|---|---|---|
| SmolVLA (dual) | wrist + overhead | 0.005 | 0.11 | ~10.4h | ~1.7B |
| SmolVLA (wrist) | wrist only | 0.006 | 0.11 | ~10h | ~1.7B |
| ACT (dual) | wrist + overhead | 0.052 | 3.92 | ~10.5h | 52M |

Loss values are not directly comparable across architectures (different loss formulations). SmolVLA converges to ~10x lower loss and ~35x lower gradient norms, expected given the pretrained VLM backbone vs ACT training from scratch with only ResNet18 features.

The wrist-only SmolVLA converges slightly higher than dual (0.006 vs 0.005) — the overhead camera provides marginal benefit for this constrained workspace.

### Next steps

- Eval all three on the real robot to compare success rates
- SmolVLA wrist-only is interesting: if comparable, it simplifies the hardware setup to a single camera
