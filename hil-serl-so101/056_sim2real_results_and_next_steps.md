# Devlog 056: Sim2Real Results and Next Steps

## Current Status

The seg+depth DrQ-v2 policy (trained in pick-101 MuJoCo, 70% sim success) is now running on the real robot with correct observation format.

### Issues Fixed
1. **Coordinate transform** (devlog 054): Added `--mujoco_mode` for MuJoCo-trained policies
2. **Observation shape** (devlog 055): Use `np.stack` for `(3, 2, 84, 84)`, batch dim only
3. **Double normalization** (devlog 055): Remove `/255` pre-normalization, encoder handles it

### Current Behavior
- Robot actively attempts to grasp the cube (not just outputting constant actions)
- Policy responds to visual input (segmentation + depth)
- Gripper opens/closes based on policy output

### Remaining Gaps
The policy attempts grasping but doesn't reliably succeed. Potential causes:

| Gap | Sim | Real |
|-----|-----|------|
| Segmentation quality | Perfect masks | EfficientViT predictions (noise, edges) |
| Depth | MuJoCo depth buffer | Depth Anything V2 disparity |
| Lighting | Constant | Variable |
| Camera pose | Fixed | Slightly different |
| Robot dynamics | Idealized | Real servos, backlash |
| Action execution | Instant | ~100ms latency |

## Options Going Forward

### Option A: More Annotation + Calibration

**Approach**: Improve the sim-to-real gap through better data and calibration.

**Tasks**:
1. Collect more segmentation training data with varied lighting/positions
2. Fine-tune EfficientViT on larger/cleaner dataset
3. Calibrate camera intrinsics/extrinsics to match sim
4. Tune depth normalization to match sim depth statistics
5. Adjust action scaling, height offsets empirically
6. Domain randomization in sim (lighting, textures, camera noise)

**Pros**:
- No changes to training pipeline
- Can improve multiple policies (RGB, seg+depth)
- Better understanding of perception gaps

**Cons**:
- Manual, iterative process
- May hit ceiling on sim2real gap
- Perception improvements may not fix dynamics mismatch

**Effort**: ~1-2 weeks of iteration

### Option B: HIL-SERL (Human-in-the-Loop SERL)

**Approach**: Fine-tune the policy on real robot with human corrections.

**How it works**:
1. Start with sim-pretrained policy (current seg+depth checkpoint)
2. Run policy on real robot
3. Human provides corrections when policy fails (teleoperation takeover)
4. Policy learns from both autonomous rollouts and human interventions
5. Iterate until reliable grasping

**Tasks**:
1. Set up SERL infrastructure (replay buffer, training loop)
2. Implement human intervention interface (keyboard/spacemouse takeover)
3. Collect ~50-100 real robot episodes with interventions
4. Fine-tune policy with RLPD (reinforcement learning from prior data)

**Pros**:
- Directly addresses sim2real gap through real experience
- Human corrections provide high-quality demonstrations
- Policy adapts to real robot dynamics
- State-of-the-art approach for robot learning

**Cons**:
- Requires real robot time (~2-4 hours of data collection)
- More complex infrastructure (online RL loop)
- Risk of forgetting sim knowledge if not careful

**Effort**: ~1 week setup + ~1 week data collection/training

### Option C: Hybrid Approach

1. Quick calibration pass (1-2 days):
   - Verify camera alignment
   - Tune action scale and height offset
   - Check depth range matches sim

2. If still <50% success, switch to HIL-SERL:
   - Use calibrated setup for data collection
   - Fine-tune with human interventions

## Recommendation

**Start with Option C (Hybrid)**:

1. **Immediate** (today):
   - Record videos of attempts, analyze failure modes
   - Quick parameter tuning (action_scale, height_offset)

2. **Short-term** (1-2 days):
   - Compare real vs sim observations side-by-side
   - Identify largest perception gaps

3. **If needed** (next week):
   - Set up HIL-SERL infrastructure
   - Collect real robot fine-tuning data

The sim-pretrained policy provides a strong prior. HIL-SERL can efficiently close the remaining gap with minimal real robot data.

## References

- [SERL Paper](https://serl-robot.github.io/): Sample-Efficient Robotic Reinforcement Learning
- [HIL-SERL Paper](https://hil-serl.github.io/): Human-in-the-Loop extension
- pick-101 training config: `drqv2_lift_seg_depth_v19.yaml`
