# Devlog 016 — EE trajectory FK validation (k crate vs placo)

## Context

Plan 003 proposes end-effector trajectory visualization using forward kinematics
from URDF files. Before implementing, we needed to validate that a pure-Rust FK
library produces results matching lerobot's placo-based FK pipeline.

## Test setup

- **Dataset:** `gtgando/so101_pick_place_10cm_v3` (v2.1, 75 episodes, 30fps)
- **URDF:** `so101_new_calib.urdf` from TheRobotStudio/SO-ARM100
- **Rust crates:** `k` 0.32 + `urdf-rs` 0.8
- **Python baseline:** `placo` via lerobot's `RobotKinematics` wrapper

## Dataset schema findings

`observation.state` is a `FixedSizeList<Float32>[6]` in parquet with columns:
```
shoulder_pan.pos, shoulder_lift.pos, elbow_flex.pos,
wrist_flex.pos, wrist_roll.pos, gripper.pos
```

Values are in **degrees** (matching lerobot's motor norm mode). Must convert to
radians before feeding to `k` crate (same as lerobot does for placo internally).

## URDF kinematic chain

`k::Chain::from_urdf_file()` correctly parses the SO101 URDF and builds:
```
root(Fixed) -> shoulder_pan(Rot) -> shoulder_lift(Rot) -> elbow_flex(Rot)
  -> wrist_flex(Rot) -> wrist_roll(Rot) -> gripper_frame_joint(Fixed)
```

`k::SerialChain::from_end("gripper_frame_link")` gives DOF=5 (the 5 arm
revolute joints, excluding gripper which branches to moving_jaw).

Joint names in the URDF match the parquet column names after stripping `.pos`
suffix: `shoulder_pan`, `shoulder_lift`, `elbow_flex`, `wrist_flex`, `wrist_roll`.

## Cross-validation results

| Frame | placo EE (x, y, z) | k crate EE (x, y, z) | Max component diff |
|-------|--------------------|-----------------------|-------------------|
| 0     | [0.2054, -0.0076, 0.0350] | [0.2054, -0.0076, 0.0350] | 0.035mm |
| 299   | [0.2157, -0.0025, 0.0382] | [0.2157, -0.0027, 0.0381] | 0.18mm  |

Sub-millimeter agreement. The frame-299 diff is larger only because the joint
values were rounded when printed; full-precision values would match closer.

## Trajectory stats (episode 0, 300 frames)

```
EE range: x=[0.2015, 0.3243] y=[-0.0604, 0.2316] z=[0.0086, 0.1953]
EE span:  dx=0.1228m  dy=0.2920m  dz=0.1866m
```

Reasonable for a pick-and-place task on the SO101's ~30cm workspace.

## Joint limit issue

Real robot data can slightly exceed URDF joint limits (e.g., shoulder_lift at
-102.9 deg vs URDF limit of +/-100 deg). `k::SerialChain::set_joint_positions()`
rejects out-of-limit values. Use `set_joint_positions_unchecked()` instead —
the FK math is valid regardless of limits.

## Implementation decision

**Use `k` + `urdf-rs` (pure Rust).** No need for placo/Python FFI.

Rationale:
- FK results match placo to <0.2mm — more than sufficient for trajectory viz
- No Python runtime dependency in a Rust desktop app
- `k` is actively maintained (v0.32, last commit Dec 2024, OpenRR ecosystem)
- Lightweight: no native deps, uses nalgebra for math
- Same API pattern as placo: load URDF -> set joints -> get transform

## Dependencies to add

```toml
k = "0.32"          # FK/IK from URDF kinematic chains
urdf-rs = "0.8"     # URDF parsing (transitive dep of k, but explicit for clarity)
```

`nalgebra` comes in as a transitive dependency of `k`.

## Implementation details

### New source files

**`src/trajectory.rs`** (219 lines):
- `RobotKinematics` — wraps `k::SerialChain` for FK. Loads URDF via
  `k::Chain::from_urdf_file()`, builds serial chain to EE frame. Input is joint
  angles in degrees (converted to radians internally). Uses
  `set_joint_positions_unchecked()` to handle out-of-limit real robot data.
- `load_episode_states()` — reads `observation.state` column from parquet using
  arrow's `FixedSizeListArray` / `Float32Array` downcasting. Returns per-frame
  joint vectors.
- `episode_data_path()` — builds parquet path from dataset root, episode index,
  and chunk size (v2.1 `data/chunk-NNN/episode_NNNNNN.parquet` pattern).
- `TrajectoryCache` — LRU cache (capacity 100) keyed by episode index. Same
  pattern as `DecodeLruCache` in cache.rs.
- `discover_urdf()` — searches `<dataset>/robot.urdf` then
  `~/.config/lerobot-explorer/robots/<robot_type>.urdf`.

**`src/trajectory_view.rs`** (210 lines):
- `OrbitCamera` — azimuth + elevation angles + zoom factor. Default: 45 deg
  azimuth, 30 deg elevation. Drag to orbit, scroll to zoom.
- `show_trajectory_3d()` — renders trajectory as 3D projected lines via
  `ui.painter()`. Orthographic projection with orbit rotation (rotate around Z
  for azimuth, tilt around X for elevation). Auto-zoom on first render to fit
  trajectory extent.
- Live playhead: when `current_frame` is provided, past trajectory is drawn
  bright with gradient, future is dimmed (gray, thinner). White dot with accent
  ring marks current EE position. Frame counter and EE xyz displayed as overlay.
- XYZ axis indicator in bottom-left corner (R/G/B for X/Y/Z).
- Time-gradient coloring: dim gray -> accent color along trajectory.

### Modified files

- `dataset.rs` — `DatasetInfo` extended with `robot_type: Option<String>` and
  `state_names: Vec<String>` parsed from info.json features.
- `app.rs` — new fields: `robot_kinematics`, `trajectory_cache`, `orbit_camera`,
  `show_trajectory`. Kinematics loaded during `load_dataset()` via URDF
  discovery. Trajectory panel added to both grid mode (dedicated right
  `SidePanel`) and single-view mode (below info panel).
- `ui/panels.rs` — `show_trajectory_panel()` method. Gets current frame from
  GridView pane (grid mode) or App playback state (single mode). Loads parquet
  and computes FK on cache miss, then renders via `show_trajectory_3d()`.
  View menu has "EE Trajectory" checkbox. Footer hints updated.
- `ui/input.rs` — `T` key toggles trajectory panel.

### URDF setup

SO101 URDF placed at `~/.config/lerobot-explorer/robots/so101_follower.urdf`
(copied from `hil-serl-so101/models/so101_new_calib.urdf`). The robot_type
field in info.json (`"so101_follower"`) maps to the filename.

## 3D depth cue improvements

The initial trajectory plot was just a line in space — hard to perceive depth
with orthographic projection and no spatial references. Options considered:

1. **Ground plane grid + drop lines** — faint grid at z=min gives a "floor".
   Vertical dashed lines from trajectory samples to the grid anchor height.
   Strongest single depth cue. **Implemented.**

2. **Shadow projection** — project the trajectory onto the ground plane as a
   faint gray line. Gives a top-down silhouette alongside the 3D view, making
   horizontal motion immediately readable. **Tried and removed** - added
   visual noise without much depth benefit compared to drop lines.

3. **Perspective projection** — replace orthographic with mild perspective
   (divide by depth). Adds foreshortening. Trade-off: harder to judge distances
   accurately. Not implemented yet.

4. **Depth-based rendering** — vary line alpha/thickness by camera depth.
   Cheap but effective for dense trajectories. Not implemented yet.

5. **Bounding box wireframe** — faint 3D wireframe cube around trajectory
   extent. Gives edge references. Not implemented yet.

## Grid overlay rendering states

Two distinct rendering modes for multi-trajectory overlay in grid view:

**No selection** (no panes clicked):
- All trajectories show past/future color split (accent vs dimmed)
- Playback progress visible across all episodes simultaneously
- No drop lines, playhead markers, or frame labels (clean bundle view)

**Has selection** (one or more panes clicked):
- Selected trajectories: full treatment (past/future split, drop lines,
  playhead marker, frame counter label). Drawn on top.
- Non-selected trajectories: dim accent lines only. Drawn behind.

Single-view mode always acts as "selected" for the current episode.

## Multi-robot support

### Architecture

The trajectory system now supports any robot without code changes:

1. **URDF discovery**: `<dataset>/robot.urdf` → `~/.config/lerobot-explorer/robots/<robot_type>.urdf`
2. **EE frame auto-detection**: finds the deepest leaf link in the URDF chain
   (no hardcoded frame name)
3. **State index extraction**: parses `state_names` from info.json, selects
   indices ending in `.pos` and not starting with `gripper`
4. **v3.0 parquet support**: shared files filtered by `episode_index` column

### SO101 (verified working)

- `robot_type`: `"so101_follower"`
- `observation.state`: 6 values, all `.pos` (contiguous)
- State pos indices: `[0, 1, 2, 3, 4]` (gripper at index 5 excluded)
- URDF: `so101_new_calib.urdf` from TheRobotStudio/SO-ARM100
- EE frame: `gripper_frame_link` (auto-detected as deepest leaf)
- DOF: 5, FK cross-validated against placo to <0.2mm

### OpenArm v10 (visually validated)

- `robot_type`: `"openarm_follower"`
- `observation.state`: 24 values, interleaved pos/vel/torque per joint
- State pos indices: `[0, 3, 6, 9, 12, 15, 18]` (7 joints, gripper excluded)
- URDF source: https://github.com/enactic/openarm_description
- DOF: 7
- Joint names: `joint_1` through `joint_7` (renamed from `openarm_left_joint1`)
- EE frame: `openarm_left_hand_tcp` (auto-detected as deepest leaf)
- FK cross-validated against placo with bimanual left arm URDF:
  placo=[0.2209, 0.1973, 0.4114], k=[0.2209, 0.1973, 0.4114], diff <0.02mm

**URDF generation process:**

xacro requires `ament_index_python` (ROS2). Worked around by monkey-patching
the module in Python to resolve `$(find openarm_description)` locally:

```python
import sys, types
ament_pkg = types.ModuleType('ament_index_python')
ament_packages = types.ModuleType('ament_index_python.packages')
def get_package_share_directory(pkg):
    return '/tmp/openarm_description'  # cloned repo path
ament_packages.get_package_share_directory = get_package_share_directory
sys.modules['ament_index_python'] = ament_pkg
sys.modules['ament_index_python.packages'] = ament_packages

import xacro
doc = xacro.process_file('urdf/robot/v10.urdf.xacro',
    mappings={'arm_type': 'v10', 'bimanual': 'true'})
```

**Critical: must use `bimanual:=true`, not `bimanual:=false`.**

The dataset was recorded with a single left arm from a bimanual setup
(`left_only: true` in the recording config). The bimanual URDF includes:
- Body frame mount: `xyz="0 0.031 0.698" rpy="-1.5708 0 0"` (arm mounted
  70cm up on body, rotated -90 deg around X)
- Joint 2 kinematic offset: `rpy="-1.5708 0 0"` (only applied in bimanual
  mode, changes the shoulder geometry significantly)
- Joint 7 axis: `xyz="0 -1 0"` (reflect=-1 for left arm)

Without the bimanual config, the FK produces a 32cm Z range where the real
robot only moves ~12cm in Z. The body mount rotation and joint2 offset are
critical for correct FK.

**Post-processing the generated URDF:**

1. Extract only left arm chain (remove right arm links/joints)
2. Strip visual/collision meshes (not needed for FK)
3. Rename joints: `openarm_left_jointN` → `joint_N` (match lerobot state names)
4. Rename links: `openarm_left_linkN` → `link_N`
5. Place at `~/.config/lerobot-explorer/robots/openarm_follower.urdf`

**Failed approaches:**
- Hand-built URDF from YAML configs: wrong because kinematics_offset was
  missing (only applies in bimanual mode)
- `bimanual:=false` xacro generation: missing body mount and joint2 offset
- `no_prefix:=true`: changes reflect logic, produces different joint naming
- Adding a 180-deg base rotation as a hack: wrong approach — the body mount
  RPY from the bimanual config is the correct transformation

### Adding a new robot

1. Get the robot's URDF (from manufacturer, ROS package, or hand-build from
   kinematic parameters)
2. Place at `~/.config/lerobot-explorer/robots/<robot_type>.urdf`
   where `<robot_type>` matches the `robot_type` field in the dataset's
   `meta/info.json`
3. Ensure joint names in the URDF match the `.pos` column base names in the
   dataset's `observation.state.names` (e.g., `joint_1` matches `joint_1.pos`)
4. The app auto-detects the EE frame (deepest leaf link) and extracts the
   correct state indices from the feature names
