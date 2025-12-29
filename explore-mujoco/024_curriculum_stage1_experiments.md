# Devlog 024: Curriculum Stage 1 Experiments

## Overview

Training the SO-101 arm to lift a cube using curriculum learning. Stage 1 starts with the cube already grasped - the agent only needs to learn to lift.

## Configuration

```yaml
env:
  max_episode_steps: 100
  action_scale: 0.02        # 2cm per step max
  lift_height: 0.08         # 8cm target
  hold_steps: 10            # Must hold for 10 steps
  curriculum_stage: 1       # Cube starts grasped
  lock_wrist: true          # Joints 3,4 locked for top-down orientation
```

## Experiment 1: v7 Reward

**Run**: `runs/lift_curriculum_s1/20251229_120848/`
**Eval video**: `runs/lift_curriculum_s1/20251229_120848/eval_200000_combined.mp4`

### Reward Structure (v7)

| Component | Condition | Value |
|-----------|-----------|-------|
| Reach | Always | `1.0 - tanh(10 * distance)` |
| Push-down penalty | `cube_z < 0.01` | `-(0.01 - z) * 50` |
| Drop penalty | Lost grasp | `-2.0` |
| Grasp bonus | Grasping | `+0.25` |
| Binary lift | `cube_z > 0.02` | `+1.0` |
| Success | Held at target | `+10.0` |

### Results

- **Training**: 200k steps
- **Final eval**: 0% success rate
- **Behavior**: Agent lifted cube to z=0.037m and stopped
- **Mean reward**: ~215 (high reward despite failure)

### Analysis

Reward hacking - the binary lift bonus at z>0.02 provides no gradient to lift higher. Agent maximizes reward by:
1. Maintaining grasp (+0.25/step)
2. Staying above 0.02m (+1.0/step)
3. Staying close to cube (reach reward ~0.9/step)

Total: ~2.15/step × 100 steps = 215 reward without actually lifting to target.

## Experiment 2: v9 Reward

**Run**: `runs/lift_curriculum_s1/20251229_131515/`
**Eval video**: `runs/lift_curriculum_s1/20251229_131515/eval_200000_combined.mp4`

### Reward Structure (v9)

| Component | Condition | Value |
|-----------|-----------|-------|
| Reach | Always | `1.0 - tanh(10 * distance)` |
| Push-down penalty | `cube_z < 0.01` | `-(0.01 - z) * 50` |
| Drop penalty | Lost grasp | `-2.0` |
| Grasp bonus | Grasping | `+0.25` |
| **Continuous lift** | Grasping | `lift_progress * 2.0` where `lift_progress = (z - 0.015) / (0.08 - 0.015)` |
| Binary lift | `cube_z > 0.02` | `+1.0` |
| Success | Held at target | `+10.0` |

### Results

- **Training**: 200k steps
- **Final eval**: 0% success rate
- **Behavior**: Agent lifted cube to z=0.068-0.073m (close but not 0.08m)
- **Entropy**: Collapsed to 0.0007 (policy too deterministic)
- **Observation**: Arm trembles/oscillates at ~0.07m instead of pushing higher

### Analysis

Better than v7 - the continuous lift gradient helped reach higher. But:

1. **Weak gradient near target**: At z=0.07m, lift reward = 0.85 × 2.0 = 1.69. At z=0.08m, lift reward = 2.0. Difference is only 0.31 reward.

2. **Entropy collapse**: Policy became deterministic too early, stopped exploring the small additional reward.

3. **Kinematic verification**: IK test confirmed arm CAN reach 0.08m with locked wrists (error <0.5mm). The issue is training, not kinematics.

```
IK Reachability Test at x=0.25, y=-0.015:
  Target z=0.05: actual z=0.0497, error=0.0004
  Target z=0.06: actual z=0.0597, error=0.0004
  Target z=0.07: actual z=0.0697, error=0.0004
  Target z=0.08: actual z=0.0797, error=0.0004
  Target z=0.10: actual z=0.0997, error=0.0003
```

## Trembling Behavior Analysis

Detailed step-by-step logging revealed the cause of failure:

```
step=  9: z=0.0815  act=[...+0.28...]  ← reached target!
step= 10: z=0.0820  act=[...-0.81...]  ← immediately pushed DOWN
step= 11: z=0.0752  act=[...+0.45...]  ← fell back
step= 12: z=0.0752  ...
```

**Key finding**: The agent DID reach 0.08m at steps 9-10, but immediately oscillated back down.

The z-action alternates wildly between positive and negative:
- +0.28 → -0.81 → +0.45 → +0.11 → -0.40 → +0.35 → ...

**Root cause**: The reward difference between 0.07m and 0.08m is too small (~0.31). The policy learned to oscillate around ~0.07m rather than hold at 0.08m. Success requires holding for 10 consecutive steps at target, but the agent never discovered the +10 success bonus because it couldn't stay at 0.08m long enough.

## Experiment 3: v10 Reward

**Run**: `runs/lift_curriculum_s1/20251229_145211/`
**Eval video**: `runs/lift_curriculum_s1/20251229_145211/eval_200000_combined.mp4`

### Reward Structure (v10)

v10 = v9 + target height bonus. Only change is +1.0 when z is within 5mm of target.

| Component | Condition | Value |
|-----------|-----------|-------|
| Reach | Always | `1.0 - tanh(10 * distance)` |
| Push-down penalty | `cube_z < 0.01` | `-(0.01 - z) * 50` |
| Drop penalty | Lost grasp | `-2.0` |
| Grasp bonus | Grasping | `+0.25` |
| Continuous lift | Grasping | `lift_progress * 2.0` |
| Binary lift | `cube_z > 0.02` | `+1.0` |
| **Target bonus** | `abs(z - 0.08) < 0.005` | `+1.0` |
| Success | Held at target | `+10.0` |

### Results

- **Training**: 200k steps
- **Rollout success**: 13%
- **Final eval**: 0% success rate
- **Behavior**: Agent lifts to z=0.075-0.077m with reduced oscillation (still minor twitching)
- **Mean reward**: ~431

| Episode | Final z | Success |
|---------|---------|---------|
| 1 | 0.076 | ❌ |
| 2 | 0.078 | ❌ |
| 3 | 0.077 | ❌ |
| 4 | 0.075 | ❌ |
| 5 | 0.077 | ❌ |

### Analysis

**Progress**: The target bonus reduced oscillation severity. Actions are smaller magnitude (±0.01 to ±0.1) but still show minor twitching around ~0.076m:

```
step= 80: z=0.0756, act=[+0.01,+0.01,-0.00,-0.11]
step= 81: z=0.0756, act=[+0.00,-0.00,+0.00,-0.11]
step= 82: z=0.0756, act=[+0.00,+0.00,+0.00,-0.11]
...stable hold at 0.0756...
```

**New problem - Reward hacking loophole**:

- Target bonus zone: `abs(z - 0.08) < 0.005` = 0.075 to 0.085
- Success criteria: `cube_z > lift_height` = z > 0.08

Agent parks at z=0.0755-0.0765 which:
- ✅ Gets the +1.0 target bonus (inside 0.075-0.085 range)
- ❌ But doesn't trigger `is_lifted` (needs z > 0.08 strictly)

The agent found a local optimum: hover at the lower edge of the target bonus zone without actually crossing the success threshold.

### Fix Options

1. **Shift target bonus zone higher**: Only give bonus when z > 0.08 (e.g., 0.08 < z < 0.09)
2. **Lower success threshold**: Change `is_lifted` to z >= 0.075 to match target zone
3. **Asymmetric bonus**: More reward for z > 0.08 than z < 0.08

## Key Learnings

1. **Binary rewards cause plateaus**: v7's binary lift bonus gave no incentive to go higher than 0.02m
2. **Gradient strength matters**: v9's 2.0x multiplier was too weak near the target
3. **Entropy collapse is a training failure mode**: Policy stops exploring before finding optimal behavior
4. **Always verify kinematics**: Confirmed the arm can physically reach the target - eliminates one hypothesis
5. **Target bonus reduced oscillation**: v10's target zone bonus reduced wild oscillation to minor twitching
6. **Reward zone must align with success criteria**: Target bonus zone (0.075-0.085) let agent exploit the lower edge without crossing success threshold (z > 0.08)
