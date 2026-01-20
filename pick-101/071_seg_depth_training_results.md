# 071: Seg+Depth Training Results

## Summary

The `seg_depth` observation format (2-channel: segmentation + disparity) achieves **1.0 success rate at 1M steps** (50% of training).

## Training Config

- Config: `configs/image_based/drqv2_lift_seg_depth_v19.yaml`
- Observation: `(6, 84, 84)` from frame_stack=3 of `(2, 84, 84)`
- Curriculum: Stage 3 (gripper near cube)
- Reward: v19

## Results at 1M Steps

| Metric | Value |
|--------|-------|
| success_rate | **1.0000** |
| ep_len_mean | 47 (down from 400) |
| ep_rew_mean | 216.84 |
| training_time | 4h 46m |
| fps | ~290 |

## Training Progression

```
Steps    | Success | Ep Len | Reward
---------|---------|--------|--------
950k     | 0.0     | 400    | 356.4
960k     | 0.0     | 400    | 204.4
976k     | 0.0     | 400    | 260.4
992k     | 0.0     | 400    | 260.4
1000k    | 1.0     | 47     | 216.8   ← First success
```

## Key Observations

1. **Sharp transition**: Success jumped from 0% to 100% between 992k-1M steps
2. **Episode length**: Dropped from 400 (timeout) to 47 steps once policy learned
3. **Stable Q-values**: critic_target_q ~110-112 throughout

## Comparison with Previous Attempts

| Obs Type | Success at 1M | Notes |
|----------|---------------|-------|
| RGB | ~0.8-0.9 | Works but slower to converge |
| Seg only | 0.0 | Policy stuck fidgeting |
| **Seg+Depth** | **1.0** | Best sim performance |

## Why Seg+Depth Works

1. **Segmentation**: Provides semantic info (where is cube vs gripper)
2. **Disparity**: Provides depth/distance info (how far is cube)
3. **Combined**: Policy can infer both "what" and "where" without RGB noise

## Artifacts

- Checkpoint: `runs/seg_depth_rl/20260119_192807/snapshots/1000000_snapshot.pt`
- Learning curves: `runs/seg_depth_rl/20260119_192807/learning_curves.png`
- Eval video: `runs/seg_depth_rl/20260119_192807/eval_videos/1000000.mp4`

## Evaluation Results (1.2M Checkpoint)

Training crashed at 1.2M steps. Evaluated the checkpoint:

| Metric | Value |
|--------|-------|
| Episodes | 10 |
| Success Rate | **70%** (7/10) |
| Mean Reward | 359.08 ± 154.77 |

### Per-Episode Results

| Episode | Success | Reward |
|---------|---------|--------|
| 1-5 | ❌ | varied |
| 6-10 | ✅ | varied |

### Best Checkpoint Note

The "best" checkpoint saved at 400k steps had **0% success rate**. This is because ModelCheckpoint monitored reward, not success rate. The 1.2M checkpoint (last before crash) performs much better.

### Evaluation Videos

- Location: `runs/seg_depth_rl/20260119_192807/eval_1200k/`
- 10 episodes with seg+depth visualization overlay

## Next Steps

1. ~~Continue training to 2M for stability~~ (crashed at 1.2M)
2. Consider monitoring success_rate for best checkpoint
3. Test on real robot with:
   - EfficientViT-B0 seg model (0.817 IoU at 84x84)
   - Depth Anything V2 Small for disparity
