# Next Agent Prompt

## Context
Debugging DrQ-v2 policy action saturation on SO-101 real robot. Policy outputs ±1 saturated actions despite state values being within expected ranges.

## What Was Done
See `devlogs/023_drqv2_action_saturation_debug.md` for full details. Key fixes applied:
1. Wrist_roll sign flip (index 4 and 10 negated)
2. Gripper converted to radians [-0.17, 1.75] instead of [-1, 1]
3. MuJoCo FK for ee_pose instead of LeRobot/Pinocchio

## Current Issue
Actions still saturated: `[-1.0, 1.0, 1.0, -0.83]` causing robot to move away from cube.

## Most Likely Remaining Issues
1. **ee_euler computation mismatch** - MuJoCo produces `[-1.53, 1.37, 3.09]` but this may differ from sim training
2. **Action coordinate frame** - dx/dy/dz might be in different frames

## Key Files
- `scripts/eval_drqv2.py:233` - `normalize_state()` function
- `scripts/eval_drqv2.py:313` - `build_state_with_mujoco_fk()` function
- `/home/gota/ggando/ml/pick-101/src/envs/lift_cube_cartesian.py` - Sim env observation format
- `scripts/rl_inference.py` - Reference real robot deployment

## Next Steps
1. Compare sim observation at same joint angles to verify ee_euler matches
2. Check if action coordinate frame needs transformation
3. Verify all 18 state values match sim format exactly
