# SmolVLA Wrist-Only Retraining with Filtered Dataset

## Background

The previous wrist-only run (devlog 012) used `--exclude-cameras overhead` which only
filtered the camera at the policy level. The dataloader still decoded both cameras'
video frames, resulting in identical ~10h training time as the dual-cam run.

To fix this, we created a wrist-only copy of the dataset that physically excludes
the overhead camera, so the dataloader never touches those video files.

## Wrist-only dataset

Created `gtgando/so101_pick_place_10cm_v3_wrist` as a local-only dataset at
`~/.cache/huggingface/lerobot/gtgando/so101_pick_place_10cm_v3_wrist/`:

- Parquet data copied from the original (1.5MB)
- Wrist videos symlinked from the original (175MB)
- Overhead videos excluded entirely
- `info.json` edited to remove `observation.images.overhead` feature and set `total_videos=75`

## Config

| Param | Value |
|---|---|
| Policy | SmolVLA (fine-tuned from `lerobot/smolvla_base`) |
| Dataset | `gtgando/so101_pick_place_10cm_v3_wrist` (75 eps, 22436 frames, wrist only) |
| Cameras | wrist only |
| Optimizer | AdamW, lr=1e-4, wd=1e-10 |
| LR scheduler | CosineDecayWithWarmup (1k warmup, 30k decay) |
| Batch size | 64 |
| Steps | 20,000 |
| Save freq | 5,000 |
| num_workers | 4 |
| image_transforms | enabled |
| video_backend | pyav |

## Training

- **Duration: ~5h 38m** (vs ~10h with dual-cam data loading)
- ~1.0s/step (vs ~1.7s/step previously)
- `data_s` ~0.25s (vs ~0.95s when loading both cameras)
- Final loss: 0.006, grad norm: 0.111, lr: 2.7e-05

### Loss curve

| Step | Loss | Grad Norm | LR |
|---|---|---|---|
| 100 | 0.118 | 0.822 | 5.1e-06 |
| 19k | 0.006 | 0.113 | 3.0e-05 |
| 19.5k | 0.006 | 0.116 | 2.9e-05 |
| 20k | 0.006 | 0.111 | 2.7e-05 |

## Artifacts

- Checkpoints: `outputs/train/smolvla_so101_10cm_v3_wristonly_tb2/checkpoints/{005000,010000,015000,020000,last}/`
- TensorBoard: `outputs/train/smolvla_so101_10cm_v3_wristonly_tb2/tensorboard/`

## Key finding

Removing overhead camera videos from the dataset cut training time from ~10h to ~5.6h (44% reduction).
The previous `--exclude-cameras` approach only prevented the policy from using overhead
features but the dataloader still decoded every frame of every video. With 2 cameras,
video decoding was the bottleneck (`data_s` ~0.95s/step), and removing one camera
halved data loading time (`data_s` ~0.25s/step).

Final loss is identical (0.006) to the previous wrist-only run, confirming
the `--exclude-cameras` policy-level filtering was working correctly for model
training — it was just wasting time on unnecessary video decoding.
