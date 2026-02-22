# HIL-SERL 500 Episode Training - Performance Degradation

**Date**: 2026-02-03
**Checkpoint**: 006500 (500+ episodes)
**Final Success Rate**: ~20% (down from 60% peak)

## Training Continuation

Continued training from checkpoint 003000 (242 episodes) to 500+ episodes as recommended in devlog 080.

## Results by Checkpoint

| Checkpoint | Episodes | Success Rate | Notes |
|------------|----------|--------------|-------|
| 3000 | 242 | 60% | Previous analysis |
| 4500 | ~371 | ~55% | Starting decline |
| 5000 | ~414 | ~50% | Continued decline |
| 6000 | ~485 | **54.9%** | Peak performance |
| 6500 | ~528 | **27.9%** | Significant crash |

## Degradation Analysis

### IK Reset Heights Tested
| Target | Actual Height | Error |
|--------|---------------|-------|
| 3cm | 2.1cm | 2.3cm |
| 5cm | 3.9cm | 2.4cm |
| 7cm | 5.9cm | 2.2cm |
| 10cm | 10.9cm | 1.5cm |

Current config uses `ik_reset_ee_pos: [0.25, 0.0, 0.03]` → ~2.1cm actual.

### Potential Causes

1. **Lighting Distribution Shift**
   - Original training (checkpoint 3000): afternoon
   - Extended training: morning
   - RGB-based visual observations sensitive to lighting

2. **Low Reset Height**
   - 2.1cm actual height limits camera view of cube
   - Higher height (6cm) would give better initial view

3. **Overfitting to Early Distribution**
   - Policy learned early episodes well
   - Failed to generalize as training continued

## Learning Curves

Saved to: `outputs/hilserl_grasp_only_v2/learning_curves_500ep.png`

Shows:
- Peak success around checkpoint 6000
- Sharp decline after checkpoint 6000
- Intervention rate remained relatively stable

## Best Checkpoint

Use **checkpoint 006000** for evaluation - peak performance before degradation.

## Next Steps (feat/grasp-v3)

1. Increase IK reset height: `0.07` target → ~6cm actual
2. Record new offline dataset with improved setup
3. Ensure consistent lighting during training
4. Consider D405 depth camera for lighting-invariant observations
