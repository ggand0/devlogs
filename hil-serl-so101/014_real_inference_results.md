# Devlog 014: Real Inference Results

## Summary

Ran trained DrQ-v2 policy (v17) on real SO-101. Policy attempts grasping behavior but fails to complete the task. Sim-to-real gap is too large without additional techniques.

## What Worked

- **IK positioning**: Robot successfully moves to training initial position using damped least-squares IK
- **Reset sequence**: Matches sim training (wrist locked at [π/2, -π/2], gripper above cube)
- **Camera pipeline**: Frame capture, center crop, resize to 84x84, frame stacking
- **Control loop**: 10Hz inference with proper timing
- **Safe return**: Lifts and returns to rest on Ctrl+C

## What Didn't Work

- **Grasping behavior**: Policy moves gripper and attempts grasp motions but doesn't successfully pick up cube
- **Action magnitude**: Policy outputs seem reasonable but don't translate to precise real-world movements
- **Visual domain gap**: Real camera images differ significantly from MuJoCo renders

## Observations

1. **Wrist orientation fix**: Had to flip wrist_roll from π/2 to -π/2 to match sim static finger position
2. **Policy behavior**: Shows learned structure (moves down, closes gripper) but timing and positioning are off
3. **Gripper control**: Opens/closes but not synchronized with cube contact

## Root Causes

1. **No domain randomization**: Sim training used fixed lighting, textures, camera pose
2. **Visual gap**: MuJoCo renders vs real USB camera images
3. **Dynamics gap**: Sim physics vs real servo response/friction

## Next Steps

Two viable paths:

### Option A: Domain Randomization
- Randomize lighting, textures, camera intrinsics in sim
- Retrain policy with augmented observations
- More sample efficient but may still have residual gap

### Option B: HIL-SERL
- Use real robot for online fine-tuning
- Human provides sparse reward signal
- More reliable but requires robot uptime

## Files

- `scripts/rl_inference.py` - Main inference with video recording
- `recordings/` - Episode videos (wrist + external camera)

## Conclusion

Pipeline is complete and functional. Pure sim-to-real transfer without domain randomization or real-world fine-tuning is insufficient for this task. This is expected - the visual and dynamics gaps are significant for manipulation tasks.
