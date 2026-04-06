# Devlog 018 -- Rename to tracelr + RANGE_M100_100 norm mode investigation

## Naming journey

Went through several rounds of naming:

1. **lerobot-explorer** (original) -- functional but generic, doesn't signal quality
2. **viewbot** -- rejected, universally associated with streaming view count fraud
3. **skimlr** (skim + lr) -- pronounced "skimmer", available everywhere. Concerns:
   credit card skimmer association, describes browsing not the key feature
4. **tracelr** (trace + lr) -- pronounced "tracer", final choice. Directly describes
   the EE trajectory tracing feature. Available on GitHub, crates.io, PyPI, npm,
   tracelr.app, tracelr.com

Naming principles that emerged:
- 3 syllables max (4+ hard to say)
- Should work as a verb ("let's tracelr that dataset")
- Name based on what makes the app unique (trajectory tracing), not generic function
- Avoid names with negative associations
- A thoughtful name signals app quality

## Rename changes

Updated all references from `lerobot-explorer` to `tracelr`:
- Cargo.toml (package name + description)
- src/main.rs (clap command name + eframe window title)
- src/app.rs (window title bar, 3 places)
- src/annotation.rs (config dir path + doc comment)
- src/trajectory.rs (config dir path + doc comment)
- README.md (header, RUST_LOG, all config paths)
- CLAUDE.md (RUST_LOG command)
- configs/prompts.example.yaml (header + path comments)

Config directory changed from `~/.config/lerobot-explorer/` to `~/.config/tracelr/`.
Symlinked robots dir to preserve existing URDF files.

## SO100 dataset testing

Downloaded official `lerobot/svla_so101_pickplace` dataset (50 episodes, v3.0,
Apache 2.0 license, 2 cameras) for README screenshots. Hit several issues:

### Issue 1: v3.0 parquet uses ListArray not FixedSizeListArray

v2.1 datasets store `observation.state` as `FixedSizeList<Float32>[6]`.
v3.0 datasets use variable-length `List<Float32>`.

**Fix:** Added ListArray fallback in `load_episode_states()`. Try FixedSizeListArray
first, then ListArray.

### Issue 2: SO100 URDF EE detection includes gripper

The SO100 URDF (`so100.urdf` from TheRobotStudio/SO-ARM100) doesn't have a
`gripper_frame_link` (fixed joint) like SO101. The deepest leaf is `jaw` (child of
`gripper` revolute joint), making DOF=6 instead of 5. But we only have 5 arm joint
values from pos_indices.

**Fix:** Modified `find_deepest_leaf()` to walk up from the deepest leaf, skipping
nodes whose joint name contains "gripper", "jaw", or "finger". For SO100 this stops
at the `gripper` link (child of `wrist_roll`), giving DOF=5. For SO101 it also
stops at `wrist_roll` since `gripper_frame_joint` and `gripper` both match.

Verified both URDFs now report DOF=5 with joint names
`[shoulder_pan, shoulder_lift, elbow_flex, wrist_flex, wrist_roll]`.

### Issue 3: RANGE_M100_100 normalization (in progress)

The official dataset uses `RANGE_M100_100` norm mode (lerobot default, `use_degrees=False`).
Joint values are normalized to [-100, 100] range instead of degrees. Our existing FK
pipeline expects degrees.

**Key findings from lerobot source:**

Motor normalization modes (`motors_bus.py`):
- `DEGREES`: `degrees = (raw_position - mid) * 360 / 4095`, where mid = (min_cal + max_cal) / 2
- `RANGE_M100_100`: `norm = ((raw - min_cal) / (max_cal - min_cal)) * 200 - 100`

The calibration min/max are per-robot values recorded during physical calibration
(moving joints through their full range). They're stored in a local JSON file on the
recording machine, NOT in the dataset.

Lerobot's `kinematics.py` `forward_kinematics()` simply does `np.deg2rad(degrees)`
and feeds to placo. No offset or limit-based mapping. This means DEGREES mode values
ARE the URDF joint angles directly (calibration homing aligns motor midpoint with
URDF zero).

**Attempts so far:**

1. **Flat 1.8x conversion** (degrees = value * 360/200): Wrong. Assumes full 4095-tick
   calibration range for all joints. Each joint has a different physical range.

2. **URDF joint limit mapping** (map [-100,100] to [lower,upper] per joint): Trajectory
   shape is closer but appears mirrored. Problem: this maps norm=0 to the URDF midpoint
   `(lower+upper)/2`, but the correct mapping should be norm=0 to URDF zero (the physical
   center after homing). For symmetric limits (like shoulder_pan [-2,2]) the midpoint IS
   zero, but for asymmetric limits (like shoulder_lift [0,3.5]) the midpoint is 1.75 rad,
   not 0.

**Root cause analysis:**

Without the calibration file from the specific robot that recorded the dataset, exact
conversion from RANGE_M100_100 to degrees is impossible. The conversion requires knowing
each motor's calibrated min/max encoder values, which depend on the physical robot instance.

The info.json doesn't store the norm mode or calibration data. There's no metadata in
the dataset to distinguish DEGREES from RANGE_M100_100.

**Detection heuristic:** If all joint pos values across the episode stay within
[-100.5, 100.5], assume RANGE_M100_100. Datasets in degrees will have values outside
this range (real robot overshoot or joints with >100 deg range).

**Next steps:**
- The conversion from RANGE_M100_100 to URDF angles needs the calibration data OR
  a better approximation
- One approach: assume norm=0 maps to URDF 0 rad, and scale by half the URDF range
  on each side: `radians = norm/100 * max(abs(lower), abs(upper))`
- Or: look for calibration files on disk that match the robot_type
- Or: add a CLI flag to specify norm mode and skip auto-detection

## Files changed this session

- `src/trajectory.rs` -- ListArray support, EE detection fix, NormMode enum,
  detect_norm_mode(), forward_kinematics_range100(), joint_limits storage
- `src/trajectory_view.rs` -- (no changes)
- All files touched by rename (see above)

## Key files for reference

| What | Where |
|------|-------|
| lerobot motor normalization | /home/gota/ggando/ml/lerobot/src/lerobot/motors/motors_bus.py:790 |
| lerobot FK (placo wrapper) | /home/gota/ggando/ml/lerobot/src/lerobot/model/kinematics.py |
| SO100 follower config | /home/gota/ggando/ml/lerobot/src/lerobot/robots/so100_follower/ |
| SO100 calibration code | so100_follower.py:calibrate() |
| Feetech homing | /home/gota/ggando/ml/lerobot/src/lerobot/motors/feetech/feetech.py:283 |
| SO100 URDF | ~/.config/tracelr/robots/so100_follower.urdf |
| SO101 URDF | ~/.config/tracelr/robots/so101_follower.urdf |
| Official dataset | ~/.cache/huggingface/lerobot/lerobot/svla_so101_pickplace |

## Datasets tried for README screenshot

| Dataset | Robot | Format | Episodes | Notes |
|---------|-------|--------|----------|-------|
| `lerobot/svla_so101_pickplace` | so100_follower | v3.0 | 50 | Official, Apache 2.0, RANGE_M100_100 (not degrees), FK broken without calibration data |
| `youliangtan/so101-table-cleanup` | so101_follower | v2.1 | 80 | 4 tasks, degrees, FK works but trajectories too dense/entangled in grid view for screenshot |
| `gtgando/so101_pick_place_10cm_v3` | so101_follower | v2.1 | 75 | Own dataset, FK validated, clean trajectories but lighting not ideal |

### Not yet tried (potential candidates for future screenshot)

- `lipsop/so101-block-in-bin-100ep` -- 100 eps, block-in-bin task, 2 cameras, so101_follower
- `aaronsu11/so101_fruit` -- 200 eps, fruit sorting, 4 tasks, 2 cameras, so101_follower
- `whosricky/so101-megamix-v1` -- 400 eps, 8 tasks, 3 cameras, v3.0 (4.7 GB)
- `observabot/so101_die_mat4` -- 217 eps, 3 cameras, HEVC codec
