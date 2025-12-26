# Devlog 017: v10 Training Results and Gripper Constraint Issue

## v10 Training Results (500k steps)

```
Eval num_timesteps=500000, episode_reward=201.71 +/- 158.87
Episode length: 100.00 +/- 0.00
Success rate: 0.00%
```

### Training Trajectory

- Maintained 1-5% rollout success rate until ~300k steps (longer than v9)
- Recovered to 5% at 340k steps after initial dip
- Eventually collapsed to 0% by end

### Eval Video Analysis

| Episode | Reward | Max cube_z | Final cube_z | Behavior |
|---------|--------|------------|--------------|----------|
| 1 | 282.74 | 0.078 | 0.015 | Lifted to 96% of target, then dropped |
| 2 | 175.99 | 0.017 | 0.016 | Never grasped, gripper stayed open (~1.25) |
| 3 | 206.94 | 0.030 | 0.015 | Inconsistent, never stable grasp |

**Key observation:** Episode 1 reached z=0.078 (96% of 0.08 target!) but then dropped the cube. The stronger lift gradient worked, but the policy is unstable.

### Progress Across Reward Versions

| Version | Max cube_z achieved | Behavior |
|---------|---------------------|----------|
| v8 | 0.047 (59%) | Hover at low height |
| v9 | 0.069 (86%) | Lift and hover, plateau |
| v10 | 0.078 (96%) | Lift high but drop |

Each version gets higher but stability decreases.

## Potential Issue: Gripper Closing Constraint

In curriculum stage 1, we lock the gripper action to prevent oscillation:

```python
# From lift_cube.py step() with lock_wrist=True
if gripper_action < 0:  # Agent wants to close
    gripper_action = self._reset_gripper_action  # Use reset value instead
```

This prevents the agent from closing the gripper beyond the initial grasp position. But depending on cube angle/position, the agent may need to close MORE to maintain grip while lifting.

**Hypothesis:** The gripper constraint is too restrictive. When the cube shifts during lifting, the agent can't adjust grip strength, leading to drops.

## Critical Bug: Gripper Constraint Logic Was Backwards

**Introduced:** Commit `2aaec15` "Add curriculum learning stages for lift task" (Dec 20, 2025)

**Affected:** ALL curriculum training runs (v8, v9, v10) - 4 days of wasted training

The original constraint:
```python
if gripper_action < 0:  # Closing
    stable_gripper = self._reset_gripper_action  # Force reset value
else:  # Opening
    stable_gripper = gripper_action  # Allow it!
```

This was completely backwards:
- **Prevented** closing tighter (which maintains grip)
- **Allowed** opening freely (which drops cube)

The agent was literally being sabotaged - it could drop the cube at will but couldn't squeeze tighter to maintain grip.

**Impact on training:**
- v8: 0% success - agent couldn't maintain grip
- v9: Early 3-4% success collapsed to 0% - policy learned opening = drop
- v10: Reached 96% height then dropped - agent opened gripper mid-lift

**Fixed to:**
```python
if gripper_action > self._reset_gripper_action:  # Opening more than reset
    stable_gripper = self._reset_gripper_action  # Prevent (would drop)
else:
    stable_gripper = gripper_action  # Allow closing/maintaining
```

Now:
- Opening beyond reset → prevented (would drop cube)
- Closing tighter → allowed (maintains grip)

## Fix: Top-Down Grasp Approach (Dec 26, 2025)

The IK reset was approaching the cube from the side (forward in X direction), causing the static finger to hit the cube first and tilt it. This resulted in diagonal grasps.

**Fixed:** Changed approach to come from above (Z+0.05) then descend, so fingers straddle the cube before closing.

```python
# Before: Forward approach
target = np.array([cube_x - 0.03, cube_y, cube_z])  # Approach from behind
# Then move forward to cube_x

# After: Top-down approach
target = np.array([cube_x, cube_y, cube_z + 0.05])  # Position above
# Then descend to cube_z
```

**Result:** 10/10 grasp success rate in tests.

## Fix: Unlock Joint 3 (wrist_flex) During Training

Training with both joints [3, 4] locked resulted in 0% eval success - the arm couldn't lift high enough.

**Problem:** Joint 3 (wrist_flex) locked at 1.65 limited vertical reach.

**Fixed:** Only lock joint 4 (wrist_roll) to keep fingers horizontal, let joint 3 flex for lifting range.

```python
# Before
locked_joints=[3, 4]
ctrl[3] = 1.65  # Fixed wrist_flex
ctrl[4] = np.pi / 2

# After
locked_joints=[4]  # Only lock wrist_roll
ctrl[4] = np.pi / 2
```

## Training Results with Fixes (Dec 26, 2025)

Run: `runs/lift_curriculum_s1/20251226_025507`

```
Eval num_timesteps=500000, episode_reward=428.15 +/- 24.92
Success rate: 0.00% (during training)
```

### Eval Results (5 episodes post-training)

| Episode | cube_z | Success | Behavior |
|---------|--------|---------|----------|
| 1 | 0.075 | No | Held steady, just under threshold |
| 2 | 0.075 | No | Held steady, just under threshold |
| 3 | 0.074 | No | Held steady, just under threshold |
| 4 | 0.081 | **Yes** | Reached target |
| 5 | 0.065 | No | Held steady |

**Success rate: 20% (1/5)**

**Key improvements:**
- Grasp maintained throughout episodes (no drops!)
- Consistent lift to ~0.075 (vs previous drops)
- Episode 4 succeeded at z=0.081

The agent reliably lifts and holds at ~0.075, just barely under the 0.08 target.

## Next Steps: Stage 3 (Grasp + Lift)

Current Stage 1 trains lifting with cube already grasped. Next step is Stage 3:

| Stage | Start State | Agent Learns |
|-------|-------------|--------------|
| 1 (done) | Cube grasped | Lift |
| 3 (next) | Gripper near cube, open | Grasp + Lift |
| 0 (final) | Arm at home, cube on table | Full task |

**Plan:**
- Fine-tune from Stage 1 best model (`runs/lift_curriculum_s1/20251226_025507`)
- Use `curriculum_stage: 3` with `_reset_gripper_near_cube()`
- Agent must learn to close gripper on cube, then lift
- May need `lock_wrist: false` to allow gripper control

## Files Changed

- `envs/lift_cube.py` - Added `_reward_v10()`, fixed gripper constraint logic, top-down approach, unlock joint 3
- `configs/lift_curriculum_s1.yaml` - reward_version: v10, lock_wrist: true, action_scale: 0.005
