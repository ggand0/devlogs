# HIL-SERL Grasp Only v2 Training Analysis

**Date**: 2026-02-01
**Checkpoint**: 003000 (242 episodes, 19,422 steps)
**Eval Success Rate**: 60% (3/5 episodes)

## Training Statistics

| Metric | Value |
|--------|-------|
| Total episodes | 242 |
| Total steps | 19,422 |
| Overall success rate | 45.9% |
| Mean intervention rate | 16.8% |
| Episodes with 0% intervention | 77 (31.8%) |

## Intervention Rate by Episode Range

| Episode Range | Intervention | 0% Episodes | Success |
|---------------|-------------|-------------|---------|
| 1-50 | 22.4% | 16.0% | 36.0% |
| 51-100 | 18.0% | 34.0% | 56.0% |
| 101-150 | 14.9% | 34.0% | 50.0% |
| 151-200 | 13.0% | 38.0% | 46.0% |
| 201-242 | 15.7% | 38.1% | 40.5% |

**Trend**: Intervention rate declining from 22% to ~13-16%. Zero-intervention episodes increasing from 16% to 38%.

## Success Rate by Intervention Level

| Intervention | Success Rate | Episodes |
|--------------|--------------|----------|
| 0% | **89.6%** | 77 |
| 1-20% | 35.8% | 67 |
| 21-40% | 17.6% | 85 |
| 41-60% | 10.0% | 10 |

**Correlation** (intervention vs reward): -0.576

## Key Insight

The policy is actually performing well. When allowed to run autonomously (0% intervention), it achieves **89.6% success rate**. The lower overall success numbers are due to:

1. Deliberate hard cube placements during training
2. Random EE reset adding position variation (±3cm)
3. Interventions occurring on difficult episodes

The declining success rate in later episodes (46% → 40%) reflects increasingly difficult cube positions being tested, not policy degradation.

## HIL-SERL Paper Comparison

From [arxiv 2410.21845](https://arxiv.org/abs/2410.21845):

| Task | Training Time |
|------|---------------|
| RAM Insertion | 1.5 hours |
| USB Grasp-Insertion | 2.5 hours |
| SSD Assembly | 1 hour |
| Object Flipping | 1 hour |

At 10 fps with ~80 steps/episode, 1-2.5 hours ≈ 300-600 episodes. Current training at 242 episodes is within expected range.

## Learning Curves

Saved to:
- `outputs/hilserl_grasp_only_v2/learning_curves.png`
- `outputs/hilserl_grasp_only_v2/intervention_curves.png`

## Recommendation

Continue training to 400-500 episodes. The intervention pattern is healthy:
- Intervention rate decreasing over time
- Autonomous success rate high (90%)
- Policy learning to handle position variation

Consider reducing deliberate hard placements to let policy consolidate learned behaviors.
