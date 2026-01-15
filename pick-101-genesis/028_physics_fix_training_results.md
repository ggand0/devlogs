# Devlog 028: Physics Fix Training Results

**Date:** 2026-01-15

## Training Run

**Directory:** `runs/drqv2/20260114_140845/`
**Config:** v19 reward, 16 envs, 1M steps, physics fixes from devlog 027

## Physics Changes Applied

| Parameter | Before | After |
|-----------|--------|-------|
| DR friction min | 0.3 | 0.5 |
| Gripper kp | 500 | 1000 |
| Physics substeps | 2 | 4 |
| Eval frequency | 100k | 50k |

## Results

### Evaluation (best_snapshot.pt @ ~800k steps)

| Metric | Before (baseline) | After (physics fix) |
|--------|-------------------|---------------------|
| Success Rate | 20% | **50%** |
| Mean Reward | 2083 ± 3118 | **4357 ± 3710** |

### Episode Breakdown

| Episode | Reward | Steps | Result |
|---------|--------|-------|--------|
| 0 | 289.90 | 200 | FAIL |
| 1 | 7774.09 | 165 | SUCCESS |
| 2 | 476.14 | 200 | FAIL |
| 3 | 8320.31 | 167 | SUCCESS |
| 4 | 8306.42 | 169 | SUCCESS |
| 5 | 7766.98 | 165 | SUCCESS |
| 6 | 8100.05 | 171 | SUCCESS |
| 7 | 1325.08 | 200 | FAIL |
| 8 | 873.06 | 200 | FAIL |
| 9 | 340.58 | 200 | FAIL |

## Observations from Eval Videos

**Positive:**
- Agent consistently attempts grasp and lift motion
- Behavior looks reasonable - approaches cube, closes gripper, lifts

**Issues:**
- **Finger penetration:** Fingers visually penetrate into the cube geometry during grasp
- **Slip during lift:** Some episodes show cube slipping out despite apparent grasp
- Penetration suggests contact solver is too soft or collision geometry needs work

## Analysis

The 2.5x improvement (20% → 50%) confirms physics parameters were part of the problem. However, remaining failures appear related to:

1. **Contact solver softness:** `solimp` values (0.98, 0.99) allow significant penetration
2. **Collision geometry:** Box pads may not perfectly match visual gripper
3. **Grasp timing:** Agent may attempt lift before fully secure grasp

## Potential Next Steps

1. **Stiffen contact solver:** Reduce `solimp` values for less penetration
2. **Review collision geometry:** Check finger pad boxes vs visual mesh alignment
3. **Reward tuning:** Add contact force or penetration penalty
4. **Longer training:** 2M steps (v19 achieved 100% in MuJoCo at 2M)

## Files

- Eval videos: `runs/drqv2/20260114_140845/eval/episode_*.mp4`
- Best checkpoint: `runs/drqv2/20260114_140845/snapshots/best_snapshot.pt`
- Training logs: `runs/drqv2/20260114_140845/`
