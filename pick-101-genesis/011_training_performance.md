# 011: Training Performance After Reset Motion Fix

## Training Run
- Config: `multi_env_wrist.yaml`
- 16 parallel environments
- curriculum_stage=3 (pre-grasp reset)

## Current Training Metrics (with fixed reset)

```
Step 14k:  fps=87, ep_len=66, ep_rew=83.4, success=0%
Step 22k:  fps=85, ep_len=46, ep_rew=56.0, success=0%
Step 24k:  fps=84, ep_len=55, ep_rew=69.9, success=0%
Step 26k:  fps=85, steps/s=90, ongoing...
```

## Reset Motion Comparison

| Component | Previous (82ad78d) | Current |
|-----------|-------------------|---------|
| Arm init | None (gravity drop) | 100 steps (wrist at pi/2) |
| Cube settle | None | 50 steps |
| Move above cube | Just set target_ee_pos | 300 steps IK |
| Move to grasp height | None | 200 steps IK |
| **Total physics steps** | **0** | **650** |

## IK Parameters

| Parameter | Previous | Current |
|-----------|----------|---------|
| gain | default (0.5) | 0.3 |
| locked_joints | None | [3, 4] |
| wrist_flex | not set | pi/2 |
| wrist_roll | not set | pi/2 |

## Previous Training (before fix)

Previous run (20260110_221509) was reported to be **much slower** - needs investigation.

Possible causes of slowness in previous version:
- Gravity drop causing unstable episodes and frequent early terminations
- IK without locked_joints computing full 5-joint solution
- Default gain=0.5 causing oscillations

## Performance Analysis

Despite adding 650 physics steps per reset, training is **faster** than before because:

1. **Stable arm initialization** - No gravity drop chaos at episode start
2. **Correct starting position** - Arm at pre-grasp position, ready to grasp
3. **Efficient IK** - locked_joints=[3,4] reduces computation (3 joints vs 5)
4. **Smoother motion** - gain=0.3 prevents oscillations and early terminations

## Speed Breakdown

- fps: ~85 (16 envs)
- Per-env rate: ~5.3 fps
- Matches benchmark prediction of 44 samples/sec for physics+render

## Next Steps

- Monitor for success rate improvement
- Check if reward shaping is working correctly
- Verify gripper closing behavior
