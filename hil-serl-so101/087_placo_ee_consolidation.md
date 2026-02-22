# Consolidating feat/hil-serl improvements into so101 (placo)

**Date:** 2026-02-11
**Branch:** `fix/placo-ee-improvements` (off `so101`)
**LeRobot fork:** `/home/gota/ggando/ml/lerobot`

## Context

The `feat/hil-serl` branch accumulated 92 commits of development on top of `so101`, mixing general-purpose SO-101 improvements with HIL-SERL-specific training code (DrQ-v2, SAC, actor-learner loop, frame stacking, etc.). The goal was to extract every reusable improvement and port it back to `so101` cleanly, while keeping placo/URDF for FK/IK (the `feat/hil-serl` branch replaced placo with MuJoCo).

All 92 commits were analyzed individually and classified as REUSABLE (~45), HIL-SERL-ONLY (~47), or MIXED. Eight commits touching isolated files were cherry-picked directly. The rest were manually ported with MuJoCo code adapted to placo.

## The state caching bug

The core issue in the original `SO101FollowerEndEffector.send_action()`:

```python
# OLD (BUGGY): cached IK output as "current" position
self.current_joint_pos = target_joints  # <-- never re-read from motors!
self.current_ee_pos = target_ee_pos     # <-- diverges from reality
```

The fix: always read real motor positions via `sync_read("Present_Position")` before computing FK/IK. No cached state variables at all.

## What was ported

### Cherry-picked commits (8)

| Commit | Description |
|--------|-------------|
| `b68072b3` | Camera auto-detection when configured device fails |
| `082472a0` | Filter metadata devices in camera auto-detection |
| `53e945ef` | Image writer handles float images in [0, 255] range |
| `d30621e1` | Image writer creates parent directories before save |
| `c4f465a7` | Protobuf `>=6.31.1` for wandb compatibility |
| `e8a28501` | Policy factory preserves `input_features` when already set |
| `1fae395f` | Buffer memory leak fix (streaming approach) |
| `aa00b85c` | Buffer `dict_keys` pickle fix |

### Manual ports by file

#### `config_so101_follower.py`
Source commits: c33ecfea, c5106b02, dce36b23, 8521a714, 2912cea3

- Added `locked_joints: list[int] | None = None` (defaults to `[3, 4]` in `__post_init__`)
- Added `locked_joint_positions: dict[str, float] | None = None` (defaults to `{"3": 90.0, "4": 90.0}`)
- Added `action_scale: float = 0.02` (meters per action unit, same as sim)
- Added `debug_ik: bool = False` (verbose IK/torque/motor logging)
- `None` defaults with `__post_init__` — draccus workaround so JSON config values actually override

#### `so101_follower_end_effector.py`
Source commits: c33ecfea, c5106b02, 487620b3, 009199e4, 0001253b, 2912cea3, 84479ad1, bbf91780, 5adf90ad, dce36b23

Full rewrite of `send_action()`:
1. Always `sync_read("Present_Position")` to get real joint positions (degrees)
2. `forward_kinematics(joints_deg)` via placo to get current EE pose (4x4 transform)
3. `target_ee = current_ee + action[:3] * action_scale` with bounds clipping
4. Build desired 4x4 pose, `inverse_kinematics(joints_deg, desired_pose)` via placo
5. Enforce locked joints at target degrees from config
6. Handle gripper: support both `[-1, 1]` (SAC tanh) and legacy `[0, 2]` (teleop) formats

Other changes:
- Added `JOINT_NAMES` constant (5 arm joints, excludes gripper)
- Camera retry in `get_observation()`: 5 retries with exponential backoff, thread-dead detection and reconnect
- All verbose logging gated behind `config.debug_ik`

#### `envs/configs.py`
Source commits: d428d3da, 43e90b2c, 18753bd6, 5949f17d, db002282, 978e33e8

Added to `EnvTransformConfig`:
```python
use_ik_reset: bool = False
ik_reset_ee_pos: list[float] | None = None       # Target [x, y, z] in meters
reset_delay_s: float = 0.0                        # Post-reset pause for object placement
capture_home_on_start: bool = False                # Use current position as home
random_ee_reset: bool = False                      # Per-episode random EE offset
random_ee_range_xy: float = 0.03                   # +/-3cm for x,y
random_ee_range_z: float = 0.02                    # +/-2cm for z
normalize_images: bool = True                      # False keeps images as [0,255]
```

#### `gym_manipulator.py`
Source commits: db08d5c4, a13eff5f, 98dfded4, 608af1a9-9ddd864d (IK reset chain), 43e90b2c, b36772c4, d428d3da, 3e6fd2aa, 4b81a9f5, 5adf90ad, db002282, feb28ace, 18753bd6, 3f4e4b03, 0d155f05, 5949f17d, 4f2adc39

**Helper functions added:**
- `_IK_MOTOR_NAMES`, `_IK_CALIBRATION_PATH` — constants for IK degree clamping
- `_load_ik_calibration()` — load JSON calibration, cache globally
- `_get_degree_limits()` — compute valid degree range per joint from encoder range + calibration
- `_clamp_degrees(joints_deg)` — clamp to valid encoder range (prevents motor rejection)
- `_robust_sync_read(bus, data_name)` — retry with exponential backoff, port clearing, USB reconnection
- `_robust_sync_write(bus, data_name, values)` — same for writes
- `reset_leader_position(leader_arm, target)` — smooth leader arm reset with torque management

**Wrapper fixes:**
- `RobotEnv.__init__`: atexit handler to disable motor torque on any exit (crashes, Ctrl+C)
- `ImageCropResizeWrapper`: `normalize_images` parameter, clamp to `[0, 1]` or `[0, 255]` accordingly
- `GripperActionWrapper`: actions now `[-1, 1]` (not `[0, 2]`), auto-detects legacy format
- `reset_follower_position()`: increased to 150 steps, 25ms interval, added logging and verification
- All `play_sounds=True` changed to `play_sounds=False`

**ResetWrapper IK reset (placo-based, 4-step sequence):**

1. **Move to SAFE_JOINTS** (all zeros) — extended forward position, gripper open, wait 1.5s
2. **Set wrist to top-down** (90deg) — reads locked_joint_positions from config, wait 1.0s
3. **Move ABOVE target** (+7cm Z) — closed-loop IK via placo:
   - Read actual joints (degrees) from bus
   - FK to get current EE, compute error to target
   - Build desired 4x4 pose, IK to get target joints
   - Clamp delta to +/-10deg/step so motors can keep up
   - Clamp to valid encoder range
   - Send and wait 100ms
   - Converge within 1.5cm, stuck detection (error not improving for 3 steps)
   - Up to 50 iterations
4. **Lower to target** — same loop with actual target EE position

Optional features:
- Random XY offset per episode (`random_ee_reset`, `random_ee_range_xy`)
- Leader arm sync after reset (`reset_leader_position`)
- Object repositioning delay (`reset_delay_s`)
- Capture home on start (skip reset motion)

**`make_robot_env` wiring:**
- Passes `normalize_images` to `ImageCropResizeWrapper`
- Passes all IK reset params (`use_ik_reset`, `ik_reset_ee_pos`, `reset_delay_s`, `capture_home_on_start`, `random_ee_reset`, `random_ee_range_xy`, `random_ee_range_z`) to `ResetWrapper`

#### `camera_opencv.py`
Source commit: 98cb891e (camera parts only, skipped reward preview)

- MJPG format + buffer=1 on VideoCapture open (reduces USB bandwidth and latency)
- `last_successful_read` timestamp for watchdog
- `_read_loop`: consecutive failure counter with auto-reconnect after 10 failures
- `_force_reconnect()`: release stuck videocapture, find working camera, restart read thread
- `async_read`: watchdog detects thread stuck >0.5s, triggers force reconnect, retries once

## HIL-SERL-only commits NOT ported (~47)

These stay on `feat/hil-serl`:
- DrQ-v2 policy, frame stacking, proprioception wrapper
- SAC actor-learner loop, buffer/learner infrastructure
- Sim-to-real action transform, reward/RL wrappers
- Full proprioception mode (18-dim state)
- Image resize wrapper (non-crop)
- Checkpoint management, wandb logging

## Verification

- All 5 modified files pass `py_compile`
- `from lerobot.robots.so101_follower.so101_follower_end_effector import SO101FollowerEndEffector` — OK
- `from lerobot.scripts.rl.gym_manipulator import make_robot_env` — OK
- `from lerobot.envs.configs import EnvTransformConfig` with correct field defaults — OK
- Zero `mujoco`/`MjModel`/`mj_forward`/`_sync_mujoco`/`_compute_ik` references in robot or gym code
- Zero `self.current_ee_pos`/`self.current_joint_pos` cached state in EE controller

## Key architectural decisions

1. **placo, not MuJoCo**: `so101` uses placo/URDF for FK/IK. `RobotKinematics` works natively in degrees. No rad/deg conversion needed in gym_manipulator IK reset (placo handles it).

2. **Degrees everywhere**: SO101FollowerEndEffector configures all arm joints as `MotorNormMode.DEGREES`. Bus reads/writes are in degrees. placo FK/IK takes and returns degrees. `_clamp_degrees` validates against calibration encoder ranges.

3. **No state caching**: Every `send_action()` call reads the actual motor position. This is slower than caching but correct — the robot physically moves between calls and cached positions diverge from reality.

4. **`num_retry=3` on critical bus operations**: USB connections to Dynamixel servos can drop. All reset, IK, and leader sync operations use retries. The `_robust_sync_read/write` helpers add exponential backoff and port reconnection for the worst cases.

5. **draccus `None` default workaround**: Fields like `locked_joints` use `None` as default with `__post_init__` setting the real default. This is because draccus (the config parser) has issues with `default_factory` when merging JSON configs — it doesn't override factory defaults properly.
