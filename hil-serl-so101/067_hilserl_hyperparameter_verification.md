# HIL-SERL Hyperparameter Verification

## Summary

Verified HIL-SERL training config against original paper and LeRobot defaults. All critical parameters are correct.

## Parameter Comparison

| Parameter | Our Config | SERL Paper | LeRobot Default | Status |
|-----------|------------|------------|-----------------|--------|
| `discount` | 0.97 | 0.95 | 0.99 | OK (minor) |
| `target_entropy` | -2.0 | -dim/2 = -2.0 | auto | OK |
| `critic_target_update_weight` | 0.005 | 0.005 | 0.005 | OK |
| `actor_lr` | 3e-4 | 3e-4 | 3e-4 | OK |
| `critic_lr` | 3e-4 | 3e-4 | 3e-4 | OK |
| `temperature_lr` | 3e-4 | 3e-4 | 3e-4 | OK |
| `temperature_init` | 0.2 | ~1.0 | 1.0 | OK (intentional) |
| `utd_ratio` | 2 | 2+ | 1 | OK |
| `batch_size` | 256 | 256 | - | OK |
| `hidden_dims` | [256,256] | [256,256] | [256,256] | OK |
| `num_critics` | 2 | 2 | 2 | OK |
| `grad_clip_norm` | 40.0 | - | 40.0 | OK |
| `use_backup_entropy` | true | - | true | OK |

## Joint Locking Status

```json
"locked_joints": [],
"locked_joint_positions": {}
```

No joints are locked. All 5 arm joints + gripper are active.

## Notes

### Discount Factor (0.97 vs 0.95)
Small difference, unlikely to affect training. Both are reasonable for short-horizon manipulation tasks.

### Temperature Init (0.2 vs 1.0)
Lower temperature encourages less exploration early. Per HuggingFace docs: "starting point is 1e-2" and "setting it too high can make human interventions ineffective and slow down learning."

### UTD Ratio (2)
Update-to-data ratio of 2 enables more gradient updates per environment step, improving sample efficiency. This is standard for HIL-SERL.

## References

- [SERL GitHub Repository](https://github.com/rail-berkeley/serl)
- [HIL-SERL GitHub Repository](https://github.com/rail-berkeley/hil-serl)
- [HIL-SERL Paper (arXiv:2410.21845)](https://arxiv.org/abs/2410.21845)
- [LeRobot HIL-SERL Documentation](https://huggingface.co/docs/lerobot/en/hilserl)
- [LeRobot SAC Configuration](https://github.com/huggingface/lerobot/blob/main/src/lerobot/policies/sac/configuration_sac.py)

## Config File

`configs/reach_grasp_hilserl_train_config.json`
