# Devlog 026: v19 Training Results Analysis

## Training Run: 20260113_182223

**Config:** v19 reward, 16 envs, 1M steps, per-reset domain randomization (cube size, lighting, friction)

## Results

### Final Evaluation (1M steps)
- **Success Rate: 20%** (2/10 episodes)
- **Mean Reward: 2083.72 ± 3118.15**
- High variance indicates inconsistent grasping

### Training Progression (from TensorBoard)

| Steps | Train Success | Eval Success | Eval Reward |
|-------|---------------|--------------|-------------|
| 100k  | 0%            | 0%           | 335         |
| 200k  | 0%            | 0%           | 308         |
| 300k  | 0%            | 0%           | 824         |
| 400k  | 0%            | 0%           | 358         |
| 500k  | 0%            | 0%           | 332         |
| 600k  | 0%            | **20%**      | 3060        |
| 700k  | 0%            | 0%           | 907         |
| 800k  | 0%            | 0%           | 312         |
| 900k  | 0%            | **20%**      | 2177        |
| 1M    | 0%            | **20%**      | 2617        |

### Key Observations

1. **Training success is always 0%** - Agent never succeeds during training episodes (with exploration noise)
2. **Eval success fluctuates** - 20% at 600k, drops to 0% at 700-800k, returns to 20% at 900k-1M
3. **Reward not monotonically increasing** - Drops from 3060 at 600k to 312 at 800k
4. **Same regression pattern as v11** - Deterministic policy inconsistent while stochastic exploration occasionally works

## Comparison: v11 vs v19

| Metric | v11 (prev run) | v19 (this run) |
|--------|----------------|----------------|
| Final eval success | 0% at 300k | 20% at 1M |
| Training success | 0% | 0% |
| Regression pattern | Yes | Yes |

v19 performed slightly better (20% vs 0%) but still shows the same fundamental issue: the learned deterministic policy is unreliable.

## Config Verification

Confirmed v19 was used:
- `runs/drqv2/20260113_182223/config.yaml` shows `reward_version: v19`
- Git history shows v19 was set before training started (commit 7ab9923 on Jan 12)

## Fix: Add Reward Version Logging

Added `[Config] Reward version: {version}` log to `lift_cube_env.py` initialization so it's clear which reward is being used.

## IK Grasp Verification

Verified that the physics and grasping mechanics work correctly using `scripts/ik_grasp_multipos.py`:
- **100% success rate** (10/10) with IK controller
- Cube positions varied by ±5cm in X and Y
- All cubes lifted ~6.5cm consistently

This confirms:
- Genesis physics simulation is working correctly
- Gripper collision and grasping mechanics are functional
- The issue is with the RL policy learning, not the environment

## Next Steps

1. Investigate why training success is always 0% (exploration noise too high?)
2. Check if domain randomization is too aggressive
3. Consider longer training (v19 achieved 100% in MuJoCo at 2M steps, not 1M)
4. Review exploration noise schedule: `linear(1.0, 0.1, 500000)` - decays to 0.1 by 500k
