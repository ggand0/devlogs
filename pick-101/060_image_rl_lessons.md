# Devlog 058: Image-Based RL Lessons Learned

## Summary

Successfully trained an image-based RL agent (DrQ-v2) to solve the SO-101 lift task with **100% success rate** at 2M steps. This required iterative reward engineering from v13 to v19.

## Why Image-Based RL Needs Explicit Reward Gradients

**Q: Do we need to micro-manage the reward more for image-based RL agents?**

Not exactly "micro-manage" - more like explicitly encode gradients that state-based RL discovers automatically.

**State-based RL gets implicit gradients from direct observation:**
- `gripper_state` changes when you close → agent sees this immediately
- `gripper_to_cube` correlates with visual proximity → direct number
- Cause-effect is immediately visible in the state

**Image-based RL loses this:**
- Gripper state change = subtle pixel changes (finger moves few pixels)
- Must learn visual encoder BEFORE learning policy
- By then, already stuck in local optimum

So v17 isn't "micro-managing" - it's baking in the gradient that state-based agents get for free. The moving finger reach reward encodes: "closing gripper when near cube = good" which state-based agents discover trivially but image-based agents struggle to find.

**Alternatives to reward engineering:**
1. Auxiliary tasks: Predict gripper state from images (forces encoder to learn relevant features)
2. Pre-trained visual encoders: R3M, MVP, etc. (bring in prior knowledge)
3. Behavioral cloning warm-start: Train policy on demos first, then fine-tune with RL
4. Curiosity/intrinsic motivation: Encourage exploring novel states

But for a simple task like this, a well-designed reward (v17-v19) is often the most direct solution. The complexity should match the problem - we're not "micro-managing," we're compensating for the observation gap between state and pixels.

## Reward Evolution Timeline

| Version | Change | Result |
|---------|--------|--------|
| v13 | Base image-RL reward | 0% grasping - couldn't learn from pixels |
| v14-v16 | Action penalty fixes, grasp bonus tweaks | Still ~0% grasping |
| v17 | Per-finger reach with proximity gate | **97.5% grasping**, 10% success |
| v18 | Doubled lift coefficient + threshold ramp | Lifts to 0.06-0.10m, but 0% success (oscillation) |
| v19 | Hold count bonus | **100% success** in 19-22 steps |

## Key Insights

### 1. Grasping is Hard from Pixels

The biggest hurdle was getting the agent to grasp at all. v13-v16 all failed at 0% grasping despite various reward tweaks. The breakthrough was v17's **per-finger reach reward**:

```python
# Only activate moving finger reward when gripper is close to cube
if gripper_reach > 0.7:  # ~3cm threshold
    moving_finger_pos = self._get_moving_finger_pos()
    moving_reach = 1.0 - np.tanh(10.0 * moving_to_cube)
    reach_reward = (gripper_reach + moving_reach) * 0.5
```

This explicitly teaches "close the gripper when near the cube" - something state-based agents learn trivially from observing `gripper_to_cube` decrease as they approach.

### 2. Reaching Height ≠ Success

v18 proved that lifting higher doesn't mean success. The agent reached 0.10m but oscillated above/below 0.08m threshold, never hitting 10 consecutive steps needed for success.

**Lesson**: Reward what you actually want (sustained height), not a proxy (instantaneous height).

### 3. Hold Count Bonus is Powerful

v19's simple addition completely solved the problem:

```python
if cube_z > self.lift_height:
    reward += 0.5 * hold_count  # 0.5, 1.0, 1.5, ... 5.0
```

This creates escalating cost for dropping below threshold:
- At step 9, dipping loses +4.5 bonus AND resets cumulative progress
- Agent learns to stabilize instead of oscillate

### 4. Sample Efficiency Gap

| Approach | Steps to Solve |
|----------|----------------|
| State-based RL | ~500k |
| Image-based RL (v19) | 2M |

Image-based requires ~4x more samples even with well-shaped rewards.

### 5. Reward Magnitude Matters Less Than Gradient

v18 had higher mean rewards (1362 vs v19's 191) but 0% success. v19 succeeded with lower rewards because the gradient pointed toward the actual goal (sustained height).

## Final v19 Reward Structure

```
v19 reward = reach + grasp + lift + hold + bonuses

reach: 0-1 (per-finger with proximity gate)
grasp: +1.5 when grasping
lift:
  - continuous: lift_progress * 4.0
  - binary: +1.0 at 0.02m
  - threshold ramp: linear 0→2.0 from 0.04m→0.08m
hold:
  - target bonus: +1.0 when above 0.08m
  - hold count: +0.5 * hold_count (NEW - the key to success)
bonuses:
  - success: +10.0
penalties:
  - drop: -2.0
  - push-down: -50*(0.01-z) when z<0.01
  - action rate: -0.02*||Δaction||² during hold
```

## Remaining Challenge: Sim-to-Real

The policy works perfectly in simulation but failed on the real SO-101 robot due to:
- Visual domain gap (MuJoCo renders vs real USB camera)
- Dynamics gap (sim physics vs real servo response)

**Next steps for sim-to-real:**
1. Domain randomization in sim (lighting, textures, camera)
2. HIL-SERL fine-tuning on real robot

## X Post

v19 evaluation video: https://x.com/gtgando/status/2008205321937060211

## Conclusion

Image-based RL requires explicit reward gradients to compensate for the observation gap vs state-based RL. The gradients aren't "micro-management" - they're encoding the same information that state-based agents get for free from direct observation. With proper reward shaping (v19), image-based RL can achieve 100% success on manipulation tasks.
