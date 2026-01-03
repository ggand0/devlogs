# V15 Reward: Gripper-Open Penalty

## Problem

Training with v14 reward got stuck at a local optimum for 1.4M+ steps. The agent:
- Positioned gripper near cube (maximizing reach reward ~0.8)
- Kept gripper wide open (state 1.2-1.78) instead of closing (<0.25)
- Never attempted to grasp, just hovered near cube

Root cause: Reach reward (~1.0 max) dominates grasp bonus (+0.25). Agent can achieve high reward without learning to close gripper.

## Solution: Gripper-Open Penalty

V15 adds a penalty for keeping the gripper open too long, encouraging exploration of closing behavior.

### Parameters

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Grace period | 40 steps | ~20 agent decisions (2 sec video), enough time to position |
| Open threshold | 0.3 | Initial gripper state in stage 3, anything above is "opening more" |
| Penalty growth | 0.05 per step / 50 | Gradual ramp over 50 steps after grace period |
| Max penalty | 0.3 per step | Caps to avoid overwhelming other rewards |

### Implementation

```python
# Track consecutive steps with gripper open (more than initial 0.3)
if gripper_state > 0.3:
    self._open_gripper_count += 1
else:
    self._open_gripper_count = 0  # Reset when gripper closes

# Penalty after grace period
grace_period = 40
if self._open_gripper_count > grace_period:
    excess_steps = self._open_gripper_count - grace_period
    open_penalty = min(0.05 * excess_steps / 50, 0.3)
    reward -= open_penalty
```

### Reward Impact

Over a 200-step episode with gripper always open:

| Behavior | Total Reward | Penalty |
|----------|-------------|---------|
| Keep gripper open | 115.3 | 12.7 (~10%) |
| Close gripper | 128.0 | 0 |

The ~10% reward difference should encourage exploration of closing.

## Counter Reset Behavior

The counter resets to 0 whenever gripper closes (state < 0.3):
- Agent closes at step 35 → counter resets
- Agent opens again at step 36 → new 40-step grace period starts

**Potential exploit**: Agent learns to briefly close every ~39 steps to avoid penalty.

**Counterpoint**: Briefly closing might accidentally lead to contact/grasp. Learning precise timing to game the penalty is harder than just learning to grasp.

## V14 vs V15 Comparison

Both rewards are identical except for the gripper-open penalty:

| Component | V14 | V15 |
|-----------|-----|-----|
| Reach reward | 1.0 - tanh(10 * dist) | Same |
| Push-down penalty | -50 * (0.01 - z) if z < 0.01 | Same |
| Drop penalty | -2.0 if was_grasping | Same |
| Grasp bonus | +0.25 | Same |
| Lift progress | +2.0 * progress (if grasping) | Same |
| Binary lift bonus | +1.0 if z > 0.02 (if grasping) | Same |
| Target height bonus | +1.0 if z > lift_height | Same |
| Action penalty | During hold phase only | Same |
| **Gripper-open penalty** | None | **0-0.3 after 40 steps** |
| Success bonus | +10.0 | Same |

## Config

```yaml
# configs/drqv2_lift_s3_v15.yaml
env:
  reward_version: v15
  curriculum_stage: 3

log_eval_video_every: 100000  # Save video + debug log every 100k steps
```

## Training Command

```bash
MUJOCO_GL=egl uv run python src/training/train_image_rl.py --config configs/drqv2_lift_s3_v15.yaml
```

## Expected Outcome

The penalty should:
1. Break the local optimum where agent hovers with open gripper
2. Encourage exploration of gripper closing actions
3. Lead to accidental contact → grasp bonus → lift learning

If agent still doesn't learn to close after significant training, consider:
- Increasing penalty magnitude
- Making counter cumulative (never reset)
- Only reset counter on successful grasp (is_grasping = True)
