# Devlog 008: Post-Clipping Fix Training Results

## Background

After fixing the gripper clipping issue (devlog 007), we retrained to see if the physics fix alone would enable successful lifting.

## Physics Fix Recap

- Added box collision primitives to gripper tips
- Stiffened contact parameters: `solref="0.001 1"` (hard plastic)
- Result: Cube stays at z=0.010 instead of being pushed to z=0.007

## V4 Reward Training (Post-Fix)

### Configuration
- Reward: V4 (grasp bonus only when cube_z > 0.01)
- Physics: Fixed (stiff contacts, box collisions)
- Timesteps: 500k

### Results

| Checkpoint | Mean Reward | Success Rate | Behavior |
|------------|-------------|--------------|----------|
| 250k | 55.76 ± 13.71 | 0% | Hover ~7-11cm from cube, gripper open (1.2-1.4) |
| 500k | 6.79 ± 1.39 | 0% | Hover ~18-21cm from cube, gripper partial close (0.3-0.6) |

### Analysis

The model **regressed** from 250k to 500k:
- At 250k: Getting close to cube but not closing gripper
- At 500k: Staying far away, learning to partially close gripper

The V4 reward structure (no grasp bonus at baseline) provided insufficient incentive to close gripper when near the cube. Without the clipping exploit available, the agent couldn't find a high-reward strategy.

## V3 Reward Training (Post-Fix)

### Reward Structure Change

V4 → V3: Added standalone grasp bonus

```python
# V3: Grasp bonus always + stronger lift when grasping
if info["is_grasping"]:
    reward += 0.25  # Grasp bonus (always)
    reward += max(0, (cube_z - 0.01) * 40.0)  # 4x lift multiplier
```

### Hypothesis

With fixed physics:
1. Grasp bonus should encourage closing gripper
2. Agent can't exploit push-down (cube stays at z=0.010)
3. Only way to increase lift reward is actual lifting

### Training Run
```
Directory: runs/lift_cube/20251216_021228/
Reward: V3 (grasp bonus always)
Physics: Fixed
Timesteps: 500k
```

V4 run (failed): `runs/lift_cube/20251216_000824/`

### Results

| Checkpoint | Mean Reward | Success Rate | Gripper | Distance | Behavior |
|------------|-------------|--------------|---------|----------|----------|
| 250k | 33.00 ± 4.57 | 0% | Closing (-0.03) | Gets close (0.04m) | Contact attempts, brief cube bumps to z=0.014 |
| 500k | 15.64 ± 3.08 | 0% | Half-open (0.55-0.67) | Far (0.15-0.19m) | Retreated, no contacts |

### Analysis

Same regression pattern as V4 - model got **worse** from 250k to 500k.

At 250k: Promising signs - gripper closing, getting within 4cm, some contact (True, False), cube briefly elevated to 0.013-0.014.

At 500k: Agent learned to hover far away with half-open gripper. No contact attempts.

### Why Both V3 and V4 Regress

The agent discovers that **avoiding the cube** is a stable, low-variance strategy:
- Reach reward saturates at ~0.9 when far away
- No risk of negative outcomes from failed grasps
- Contact with one side (True, False) gives **no reward** - only full grasp (True, True) triggers bonus

The fundamental issue: **sparse grasp reward**. Agent must randomly achieve both-sided contact + closed gripper to get any grasp signal. This is too hard to discover through exploration.

## Key Insight

The clipping fix closed the physics exploit, but exposed a deeper problem: the grasp reward is too sparse. Agent must achieve:
1. Closed gripper (< 0.25)
2. Contact on static gripper side
3. Contact on moving jaw side

All simultaneously to get any grasp bonus. Single-sided contact (True, False) gives zero reward.

## Next Steps

Options to try:
1. **Partial contact reward** - Give small bonus for any cube contact
2. **Gripper closing reward** - Reward closing gripper when near cube
3. **Curriculum** - Start with cube in gripper, learn to lift first
4. **HER** - Hindsight Experience Replay for goal-conditioned learning

## Files

- `envs/lift_cube.py` - V3 reward (grasp bonus always)
- `SO-ARM100/Simulation/SO101/so101_new_calib.xml` - Gripper collision boxes
- `SO-ARM100/Simulation/SO101/lift_cube_scene.xml` - Stiff contact parameters
