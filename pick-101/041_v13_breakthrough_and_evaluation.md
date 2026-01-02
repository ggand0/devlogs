# Devlog 041: v13 Reward Breakthrough & Evaluation

## Breakthrough Results

**First successful image-based grasping at 800k steps!**

| Metric | Before (v11) | After (v13) |
|--------|--------------|-------------|
| Reward | ~320 (plateau) | 650+ |
| Success Rate | 0% (2M steps) | **10%** (800k steps) |
| Behavior | Single-finger tilt exploit | Proper two-finger grasp |

### Training Progress (v13)
```
800k steps | 3:08:40 elapsed
ep_rew_mean: 650.17
success_rate: 0.10 (10%)
New best model saved!
```

### What Fixed It
v13 gates the binary lift bonus on `is_grasping`:
```python
# v11 (exploitable):
if cube_z > 0.02:
    reward += 1.0  # Tilt exploit gets this

# v13 (fixed):
if is_grasping:
    if cube_z > 0.02:
        reward += 1.0  # Only with proper grasp
```

Reward gap: Exploit ~0.9/step vs Grasp ~2.33/step (was 1.9 vs 2.33 in v11)

## Final Training Results (2M steps)

Training completed after 2M steps (~8 hours).

### Training Summary
| Checkpoint | Reward | Success Rate | Notes |
|------------|--------|--------------|-------|
| 800k | 650.17 | 10% | **Best model saved** |
| 1M | ~400 | ~5% | Performance dropped |
| 1.5M | ~350 | 0% | Continued decline |
| 2M | ~315 | 0% | Final checkpoint |

The agent peaked at 800k and then regressed. The best model was automatically saved at the 800k peak.

## Evaluation Results

Evaluated the best checkpoint (saved at ~800k steps) with 10 deterministic episodes.

### Metrics
| Metric | Value |
|--------|-------|
| Episodes | 10 |
| Mean Reward | 632.27 ± 43.00 |
| Success Rate | 0% |

### Observed Behavior
- Agent successfully approaches the cube
- Gripper closes and grasps the cube properly
- Lifts cube to a certain height (~4-5cm)
- **Shaking behavior observed** after lifting - possible reward hacking or instability
- Does not reach the 8cm height threshold for 3+ seconds required for success

### Video Analysis
Evaluation videos saved to: `runs/image_rl/20260101_204513/eval_videos/`
- `episode_00.mp4` through `episode_09.mp4`

The high reward (~630) indicates the agent is doing meaningful work (grasping + partial lift), but the shaking behavior at height suggests:
1. Possible reward hacking (oscillating to accumulate lift reward without full success)
2. Instability in the learned policy at elevated positions
3. Action noise from stochastic policy (though eval uses deterministic actions)

## Files

- **Training run**: `runs/image_rl/20260101_204513/`
- **Best checkpoint**: `snapshots/best_snapshot.pt` (saved at 800k, 650.17 reward)
- **Config**: `configs/drqv2_lift_s3.yaml`
- **Eval videos**: `runs/image_rl/20260101_204513/eval_videos/episode_*.mp4`

## Analysis & Next Steps

### Why 0% Eval Success Despite 10% Training Success?
1. **Stochastic vs deterministic**: Training uses exploration noise; eval is deterministic
2. **Overfitting to specific initial states**: Training may have found lucky trajectories
3. **Evaluation threshold stricter**: Need 8cm for 3 full seconds

### Potential Improvements
1. **Reward shaping**: Penalize excessive gripper/cube velocity at height
2. **Longer hold requirement during training**: Force stable holds
3. **Domain randomization**: Initial cube position, lighting
4. **Curriculum adjustment**: Stage 4 with tighter gripper-cube alignment

### See Also
- [042: Sim-to-Real Transfer Plan](./042_sim_to_real_transfer_plan.md) - Hardware integration roadmap
