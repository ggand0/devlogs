# Devlog 083: Randomization Alignment with HIL-SERL Paper

**Date:** 2025-02-05

## Summary

Updated reset position randomization to match the HIL-SERL paper's settings for reproducibility.

## Change

**Previous settings:**
- `random_ee_range_xy`: ±3cm (0.03)
- `random_ee_range_z`: ±2cm (0.02)

**New settings (matching paper):**
- `random_ee_range_xy`: ±1cm (0.01)
- `random_ee_range_z`: ±1cm (0.01)

## Paper Reference

From [HIL-SERL: Precise and Dexterous Robotic Manipulation via Human-in-the-Loop Reinforcement Learning](https://arxiv.org/abs/2410.21845):

> "For each subtask, we uniformly randomized the scripted grasping poses by **1 cm in each translation dimension** to introduce variations to the policy."

## Files Updated

- `configs/grasp_only_record_v3_config.json`
- `configs/grasp_only_hilserl_train_config.json`
- `configs/grasp_only_hilserl_eval_config.json`

## Rationale

The original ±3cm XY / ±2cm Z settings were more aggressive than the paper's recommendations. While larger randomization could improve generalization, aligning with the paper first provides a reproducible baseline. If training succeeds with paper settings, we have a known-good reference point. If it fails, we can rule out randomization range as a variable.

## Notes

- The eval config has `random_ee_reset: false`, so the range values don't affect evaluation, but updated for consistency.
- Future experiments may test larger ranges (±3cm) to compare generalization.
