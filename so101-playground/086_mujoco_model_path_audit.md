# Devlog 086: `mujoco_model_path` Code Audit

**Date:** 2026-02-08

## Summary

Audited all usages of `mujoco_model_path` across the LeRobot fork (`ggand0/lerobot`, branch `feat/hil-serl`) and the config repo (`ggand0/hil-serl-so101`). Found 3 bugs — none critical enough to have blocked training (devlog 085 achieved 70% grasp success with these bugs present), but worth fixing for correctness.

## Background

`mujoco_model_path` was introduced to replace `urdf_path` for SO-101 end-effector control. MuJoCo is used purely as a kinematics solver (FK/IK), not for simulation. The evolution was:

1. URDF + placo (devlog 016) — broken state caching caused drift
2. MuJoCo IK direct, bypassing gym — worked but wasn't integrated
3. MuJoCo IK integrated into `SO101FollowerEndEffector` class — current approach

Using MuJoCo as a kinematics library is a reasonable choice: the pip package is lightweight (just C bindings), FK/IK calls are microseconds, and the same XML model used in sim training gives exact kinematic consistency on the real robot. Alternatives (pinocchio, ikpy, custom DH params) would add complexity without clear benefit.

## All Usages

### LeRobot fork — All actively load-bearing

| File | Lines | Role |
|------|-------|------|
| `robots/so101_follower/config_so101_follower.py` | 50 | Dataclass field definition |
| `robots/so101_follower/so101_follower_end_effector.py` | 71-77 | Loads MuJoCo model for FK/IK in `__init__` |
| `scripts/rl/gym_manipulator.py` | 591 | EE control detection in `step()` — gates whether policy actions reach motors |
| `scripts/rl/learner.py` | 1172, 1190-1192, 1253, 1275 | Reads from config, passes to `ReplayBuffer` for offline FK/action conversion |
| `utils/buffer.py` | 443, 520-530 | `from_lerobot_dataset()` loads MuJoCo for FK when expanding 6-dim → 18-dim state |

### Config repo (hil-serl-so101) — All active

All 21 JSON config files reference `/home/gota/ggando/ml/pick-101/models/so101/lift_cube.xml`. Two utility scripts also hardcode the path (`test_ik_heights.py`, `position_arm_center.py`).

## Bugs Found

### Bug 1: `FullProprioceptionWrapper` reads stale MuJoCo state

**Location:** `gym_manipulator.py:1903-1906`

```python
if self.use_mujoco_fk:
    robot = self.unwrapped.robot
    ee_xyz = robot.mj_data.site_xpos[robot.ee_site_id].copy()
```

This reads `robot.mj_data` without calling `_sync_mujoco()` with the current joint positions. The execution order in `step()` is:

1. `send_action()` → calls `_sync_mujoco(current_joints_rad)` then `_compute_ik()` — final `mj_data` state reflects joints at time of action
2. `_get_observation()` → reads fresh joint positions from motor bus
3. `FullProprioceptionWrapper.observation()` → reads `mj_data` which still has joints from step 1, not the fresh positions from step 2

Result: the EE position in the 18-dim proprioceptive state uses `joints_A` (from `send_action`'s `sync_read`) while `agent_pos` uses `joints_B` (from `_get_observation`'s `sync_read`). These are inconsistent within the same observation.

However, the actual staleness is small: `joints_A` and `joints_B` are separated by only ~3-5ms (IK computation ~microseconds + `sync_write` ~1-2ms + `sync_read` ~1-2ms). At STS3215 servo speeds (~360 deg/s max), 5ms = ~1.8 degrees at full speed. In practice the servos are usually near their target and moving slowly, so the delta is likely **< 0.5 degrees**, translating to **sub-millimeter EE position error** through the SO-101's kinematics.

**Severity:** Negligible in practice. The inconsistency is real but the magnitude is tiny at 10 Hz control with these servos.

**Fix:** Sync MuJoCo with current `agent_pos` before reading EE:
```python
if self.use_mujoco_fk:
    robot = self.unwrapped.robot
    joint_pos_rad = np.deg2rad(observation["agent_pos"][:robot.N_ARM_JOINTS])
    robot._sync_mujoco(joint_pos_rad)
    ee_xyz = robot.mj_data.site_xpos[robot.ee_site_id].copy()
```

### Bug 2: `KeyboardTeleopWrapper` has the same stale-state issue

**Location:** `gym_manipulator.py:2171-2178`

Same pattern — reads `robot.mj_data` without syncing. Same consequence and fix.

### Bug 3: `buffer.py` hardcodes `qpos[:6]` in action conversion path

**Location:** `buffer.py:714-719`

```python
prev_joints_rad = np.deg2rad(prev_joints)
mj_data.qpos[:6] = prev_joints_rad
```

The FK computation path (line 647) correctly uses `num_dof = len(joint_pos)` dynamically. But the action conversion path hardcodes `[:6]`. The SO-101 arm has 5 arm joints + 1 gripper in the observation. If the MuJoCo model has only 5 `qpos` entries (arm only), numpy silently writes only 5 values — the 6th (gripper degrees) is dropped. If the model has 6+ joints, the gripper position (in degrees, not radians) gets written as radians, corrupting FK.

**Severity:** Low. In practice the `lift_cube.xml` model likely handles this gracefully (numpy slice truncation), but it's inconsistent with the FK path and could break with a different model.

**Fix:** Use `min(6, mj_model.nq)` like the FK path does.

### Not bugs (investigated and cleared)

- **`gym_manipulator.py:591` `hasattr` check** — Correct. `SO101FollowerEndEffectorConfig` has `mujoco_model_path`, `SO101FollowerConfig` doesn't.
- **`gym_manipulator.py:2921` only checking `urdf_path`** — Intentional. MuJoCo robots handle EE pose internally.
- **`gym_manipulator.py:1444` `not hasattr(... 'urdf_path')`** — Falls through to joint mirroring branch, which is correct for SO-101 teleop reset.
- **`gym_manipulator.py:1871` and `1988`** — Properly dual-path: checks `urdf_path` first, falls back to `hasattr(robot, 'mj_data')`.
- **`learner.py:1190-1192` double-getattr** — Handles both flat and nested config objects correctly.
- **`locked_joint_positions` string vs int keys** — Both `so101_follower_end_effector.py:264` and `gym_manipulator.py:582` have int-then-string fallback. Works correctly.

## Impact on Training

These bugs were present during the v3_lamp training run (devlog 085) which achieved 70% grasp success. The stale-state bug (Bug 1) has only ~3-5ms of staleness within a single step, translating to sub-millimeter EE error — effectively zero impact. The buffer hardcoding (Bug 3) only affects offline-to-online conversion and likely had no effect since numpy silently truncates the slice.

None of these bugs affected training. Bug 1 is technically incorrect but the magnitude is negligible at 10 Hz with STS3215 servos. Would only matter at significantly higher control frequencies or with faster actuators.
