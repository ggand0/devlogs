# 001 — IK Reset for Data Collection

**Date**: 2026-02-14

## Problem

Vanilla lerobot `record.py` only has a timed pause (`reset_time_s`) between episodes for manual reset. For consistent pick-and-place demos we need the arm to automatically return to a start position before each episode.

## Approach

Created `scripts/record.py` — a custom recording script that reuses lerobot's `record_loop()` but replaces the between-episodes reset with a placo-based IK reset.

### Why placo/URDF instead of MuJoCo

The `fix/placo-ee-improvements` branch of our lerobot fork has `RobotKinematics` (in `lerobot.model.kinematics`) wrapping the placo library. It takes/returns degrees natively, matching the `use_degrees=True` convention. No MuJoCo dependency needed.

### 4-step IK reset sequence

From devlog 087 in hil-serl-so101 / `gym_manipulator.py` `ResetWrapper`:

1. **SAFE_JOINTS** — all 5 arm joints to 0deg (arm extended forward), gripper open at 50%. Wait 1.5s.
2. **Wrist top-down** — read current joints, set wrist_flex=90deg, wrist_roll=90deg. Wait 1.0s.
3. **Move ABOVE target** — closed-loop IK to `[0.25, 0.0, 0.14]` (target + 7cm Z).
4. **Lower to target** — closed-loop IK to `[0.25, 0.0, 0.07]`.

Steps 3-4 use the same closed-loop IK loop:
- Read real joints (degrees) from bus
- FK to get current EE position
- Compute position error
- IK to get target joints
- Clamp delta to +/-10deg per step
- Clamp to valid encoder range via `_clamp_degrees`
- Write to bus, wait 100ms
- Converge within 1.5cm or stuck detection (3 steps no improvement)
- Max 50 iterations

### After reset, before recording

- Sync leader arm to follower position (write follower's Present_Position to leader's Goal_Position)
- Wait 0.5s for leader to reach position
- Disable leader torque so user can teleoperate

## Key imports from lerobot fork

From `lerobot.model.kinematics`:
- `RobotKinematics(urdf_path, target_frame_name, joint_names)` — must pass `joint_names=list(_IK_MOTOR_NAMES)` (5 arm joints only, excluding gripper) because the URDF has 6 joints

From `lerobot.scripts.rl.gym_manipulator`:
- `_IK_MOTOR_NAMES` = `["shoulder_pan", "shoulder_lift", "elbow_flex", "wrist_flex", "wrist_roll"]`
- `_clamp_degrees(joints_deg)` — clamps to valid encoder range using calibration file

From `lerobot.record`:
- `record_loop()` — the actual teleoperation recording loop

From `lerobot.utils.control_utils`:
- `init_keyboard_listener()` — right arrow = exit early, left = rerecord, esc = stop

## Bugs fixed during implementation

1. **`hw_to_dataset_features` kwarg**: branch uses `use_video=` not `video=`
2. **`sanity_check_dataset_name` kwarg**: branch uses `policy_cfg=` not `policy=`
3. **Missing robot `id`**: must set `id="ggando_so101_follower"` / `id="ggando_so101_leader"` to find existing calibration files
4. **URDF joint count**: `RobotKinematics` defaults to all 6 URDF joints (including gripper), but IK only uses 5 arm joints — must pass `joint_names=list(_IK_MOTOR_NAMES)` explicitly
5. **URDF mesh assets**: URDF references `assets/*.stl` relative paths, so the `assets/` directory must be copied alongside the URDF

## Test run

```
IK above: stuck after 11 steps (error=0.0356m)
IK lower: converged in 2 steps (error=0.0143m)
```

The "above" step gets stuck at ~3.5cm error — good enough, the arm is roughly above target. The "lower" step converges quickly from there. The 3.5cm residual on step 3 is expected given the simplified IK (no orientation constraint enforcement beyond the locked wrist).

## Files

| File | Description |
|------|-------------|
| `scripts/record.py` | Recording script with IK reset |
| `scripts/record.sh` | Shell wrapper |
| `models/so101_new_calib.urdf` | URDF from SO-ARM100 repo |
| `models/assets/` | STL meshes for URDF (gitignored) |
| `pyproject.toml` | Added `kinematics` extra for placo |

## Dependencies on lerobot fork

Branch: `fix/placo-ee-improvements` at `../lerobot`

Key files on the branch:
- `lerobot/model/kinematics.py` — `RobotKinematics` placo wrapper
- `lerobot/scripts/rl/gym_manipulator.py` — `_clamp_degrees`, `_IK_MOTOR_NAMES`, `ResetWrapper`
- `lerobot/robots/so101_follower/config_so101_follower.py` — `SO101FollowerConfig.use_degrees`
- `lerobot/teleoperators/so101_leader/config_so101_leader.py` — `SO101LeaderConfig.use_degrees`
