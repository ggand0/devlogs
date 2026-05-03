# 025: URDF Management UX

Date: 2026-04-29
Branch: `feat/urdf-management`

## Problem

Installing a URDF for trajectory visualization required manually creating `~/.config/tracelr/robots/`, figuring out the expected filename from the dataset's `robot_type`, and copying the file in. Most users just used `--urdf` on every launch instead. The app also showed no indication that a URDF was missing -- the trajectory panel simply didn't appear.

## Changes

### Drag-and-drop URDF install

The existing `handle_dropped_files` in `ui/input.rs` only handled dataset directories. Now it checks for `.urdf` extension first and routes to a new `handle_urdf_drop()` path.

The install flow: `install_urdf()` in `trajectory.rs` copies the dropped file to the robots config dir (`~/.config/tracelr/robots/` on Linux, `~/Library/Application Support/tracelr/robots/` on macOS), creating the directory if needed. The destination filename depends on context:

- If a dataset is loaded and the source filename already starts with the `robot_type` (e.g. dropping `openarm_follower_right.urdf` on an `openarm_follower` dataset), the original name is preserved. This matters for future multi-arm support where the glob `<robot_type>*.urdf` will discover multiple arms.
- If the source filename doesn't match the robot type, it gets renamed to `<robot_type>.urdf` so discovery picks it up automatically.
- If no dataset is loaded, the original filename is kept as-is.

After copying, `reload_kinematics_from()` loads the URDF and clears the trajectory cache so the new kinematics take effect immediately.

### "No URDF" panel with helper buttons

Previously the trajectory panel was gated on `robot_kinematics.is_some()` -- it simply didn't render without a URDF. Now it shows whenever a dataset is loaded and the "EE Trajectory" checkbox is enabled.

When kinematics are missing, `show_urdf_missing_panel()` renders:
- The robot type and expected filename (e.g. `Expected: openarm_follower.urdf`)
- A hint that .urdf files can be dropped onto the app
- "Open robots folder" button -- runs `xdg-open`/`open`/`explorer` per OS on the robots config dir (creates it first if needed)
- "Browse..." button -- native file picker via `rfd` with a `.urdf` filter, same install + reload flow as drag-and-drop

The "EE Trajectory" checkbox in the View menu is now visible even without kinematics loaded, so users can toggle the panel on to discover the setup instructions.

### Helper refactoring in trajectory.rs

Extracted `robots_config_dir()` as a shared helper, used by both `discover_arms()` and `install_urdf()`. Previously the config dir path was inlined in `discover_urdf()`.

### Incidental fix

The `episode_data_path_v21` test was broken on main (missing `data_chunk_index` and `data_file_index` args after that function's signature changed in the v3 multi-file work). Fixed the test calls to pass the two extra zero args.

## Multi-arm URDF support (commit 2)

### Data model change

Replaced single `robot_kinematics: Option<RobotKinematics>` + `state_pos_indices: Vec<usize>` with `arms: Vec<ArmKinematics>` + `active_arm_index: usize`. Each `ArmKinematics` bundles a name, `RobotKinematics`, and per-arm `pos_indices`.

### Discovery: `discover_arms()`

Replaces the old `discover_urdf()`. Checks in priority order:

1. Dataset-local `robot.urdf` -- single arm, same as before
2. `<robot_type>.toml` in config dir -- explicit multi-arm config (see below)
3. Filename glob `<robot_type>*.urdf` in config dir:
   - Single match: single arm, backwards-compatible
   - Multiple matches: one arm per file, name derived from filename suffix (e.g. `openarm_follower_left.urdf` becomes arm name "left")

### TOML config override

Optional `<robot_type>.toml` in the robots config dir for explicit control:

```toml
[[arm]]
name = "Left Arm"
urdf = "openarm_follower_left.urdf"
joint_prefix = "openarm_left_"
# ee_frame = "custom_frame"

[[arm]]
name = "Right Arm"
urdf = "openarm_follower_right.urdf"
joint_prefix = "openarm_right_"
```

Parsed via serde into `RobotConfig` / `ArmConfig` structs. `joint_prefix` filters state column names by prefix. `ee_frame` overrides the auto-detected end-effector. Both are optional -- without `joint_prefix`, falls back to joint name intersection matching.

Note: `joint_prefix` is for bimanual datasets where state columns are prefixed (e.g. `openarm_left_joint_1.pos`). Single-arm datasets with generic names (`joint_1.pos`) should omit it.

### Per-arm joint matching

Two strategies for mapping URDF joints to dataset state columns:

1. **`match_pos_indices_by_joints()`** (default, used by glob path and TOML without `joint_prefix`): Iterates URDF joint names and finds matching `.pos` state columns by checking if the state column base name equals or ends with the joint name.

2. **`pos_indices_with_prefix()`** (TOML with `joint_prefix`): Filters state columns by prefix, for bimanual datasets where both arms' data is in the same state vector.

### Arm selector UI

The trajectory panel shows a `ComboBox` dropdown when multiple arms are loaded. Switching arms clears the trajectory cache and recomputes trajectories with the new arm's kinematics and pos_indices.

## Files changed

- `Cargo.toml` -- added `toml = "0.8"` dependency
- `src/trajectory.rs` -- `ArmKinematics`, `RobotConfig`/`ArmConfig` structs, `discover_arms()`, `load_single_arm()`, `load_arms_from_toml()`, `match_pos_indices_by_joints()`, `pos_indices_with_prefix()`, `RobotKinematics::joint_names()`, removed `discover_urdf()`
- `src/app.rs` -- replaced `robot_kinematics`/`state_pos_indices` with `arms`/`active_arm_index`, updated `load_dataset()`
- `src/ui/input.rs` -- `.urdf` drop detection, `handle_urdf_drop()`, `reload_kinematics_from()` updated for multi-arm model
- `src/ui/panels.rs` -- arm selector dropdown, cache clear on arm switch, `show_urdf_missing_panel()`

## Persistent arm selection (commit 3)

### Problem

Auto-detection can't distinguish left from right when both URDFs have identical joint names (`joint_1` through `joint_7`) and the dataset state columns have no arm prefix. Both arms match equally, so the app defaulted to whichever came first alphabetically.

### Solution

Persist the user's arm choice to `~/.config/tracelr/arm_preferences.json`, keyed by canonicalized dataset path, value is the arm name string:

```json
{
  "/home/user/datasets/openarm-left-demo": "left",
  "/home/user/datasets/openarm-right-demo": "right"
}
```

- **Save**: written to disk immediately when the user switches arms in the dropdown. Also updates the in-memory `arm_preferences` HashMap so subsequent dataset switches within the same session pick it up.
- **Load**: read once at app startup into the in-memory map. On dataset load, `lookup_arm_preference` canonicalizes the dataset path and checks the map. If a match is found and the arm name exists in the loaded arms, it's selected.
- **No implicit saves**: if the default arm is correct, no preference is written. Only explicit user switches trigger a save.

Matching by arm name (not index) so preferences survive URDF file reordering.

### Bug found during testing

Initial implementation only wrote to disk but didn't update the in-memory `arm_preferences` map. So preferences saved during the current session were invisible to subsequent `load_dataset` calls. Fixed by updating both disk and in-memory map on arm switch.

### Files changed

- `src/trajectory.rs` -- `arm_prefs_path()`, `load_arm_preferences()`, `save_arm_preference()`, `lookup_arm_preference()`
- `src/app.rs` -- `arm_preferences: HashMap<PathBuf, String>` field, loaded at startup, lookup in `load_dataset()`
- `src/ui/panels.rs` -- save to disk + update in-memory map on arm switch

## Testing

Tested with OpenArm datasets:
1. Removed existing URDFs from config dir to trigger "No URDF" state -- panel shows with correct robot_type and expected filename
2. Browse and drag-and-drop both install and load URDFs immediately
3. Dropping `openarm_follower_right.urdf` preserves the filename (does not overwrite `openarm_follower.urdf`)
4. "Open robots folder" opens the file manager at the correct path
5. With both `openarm_follower_left.urdf` and `openarm_follower_right.urdf` in config dir, the app shows "left" / "right" dropdown and switching produces correct trajectories for each arm
6. Creating `openarm_follower.toml` with custom arm names shows "Left Arm" / "Right Arm" in dropdown
7. TOML with `joint_prefix` produces empty pos_indices on single-arm datasets (expected -- prefix is for bimanual datasets). Removing `joint_prefix` falls back to joint name matching and works correctly
8. Arm preference persistence: select right arm on dataset A, open dataset B, reopen dataset A -- right arm is restored
9. Preference survives app restart (read from disk on startup)
