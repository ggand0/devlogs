# Devlog 020 -- Camera cycling (multi-camera phase 1)

## Goal

Let users switch between cameras in both single-video and grid modes.
Datasets typically have 2-6 cameras (wrist, top, side, etc.) but the app
was locked to one camera selected at load time (preferring "wrist").

## Design decisions

### ComboBox placement

Replaced the static "Camera: wrist" text in the right info panel with an
egui ComboBox dropdown. Considered putting it in the grid footer bar but
that gets crowded and the info panel already shows camera metadata. The
ComboBox only renders when the dataset has >1 camera; single-camera
datasets show the name as plain text.

### Deferred switch pattern

`show_info_panel()` doesn't have access to `egui::Context` (it receives
only `&mut egui::Ui`), but `switch_camera()` needs `ctx` to rebuild
VideoPlayers and the episode cache. Solved with a `pending_camera_switch:
Option<usize>` field on App. The ComboBox sets it; the `update()` loop
checks and applies it before rendering panels. Standard egui deferred
action pattern.

### Keyboard shortcuts

`C` / `Shift+C` cycle forward/backward through cameras. Fires before the
grid vs single-video keybind split so it works in both modes. Uses
`rem_euclid` for wrapping.

## Implementation

### rebuild_video_paths() extraction

Extracted the video_paths + seek_ranges construction from `load_dataset()`
into a standalone method. Before this, the path-building logic was inlined
in `load_dataset()` and not reusable. Now `load_dataset()` calls
`rebuild_video_paths()` after setting the dataset and camera index, and
`switch_camera()` reuses the same method.

### switch_camera() flow

```
switch_camera(new_index, ctx)
  |-- validate bounds, skip if same index
  |-- current_video_key_index = new_index
  |-- rebuild_video_paths()          (new paths for all episodes)
  |-- decode_cache.clear()           (LRU thumbnails are stale)
  |-- episode_cache = None           (sliding window is stale)
  |-- init_cache(ctx)                (rebuild sliding window)
  |
  |-- if grid mode:
  |     rebuild GridView with same start_episode, cols, rows
  |-- if single-video mode:
  |     re-enter video mode (new VideoPlayer for current episode)
```

Added `DecodeLruCache::clear()` -- the LRU cache stores decoded middle
frames keyed by episode index. After a camera switch every cached frame
is for the wrong camera, so clearing is mandatory.

### Grid footer updates

Added the current camera name (bright text) after the episode range in
the grid footer. Added `[C] camera` to the keybind hint string. The hint
string uses a `match (has_trajectory, has_multi_cam)` to avoid runtime
formatting for static strings.

## Files changed

- `src/app.rs` -- `rebuild_video_paths()`, `pending_camera_switch` field,
  deferred switch in update loop
- `src/cache.rs` -- `DecodeLruCache::clear()`
- `src/playback.rs` -- `switch_camera()`, `cycle_camera()`
- `src/ui/input.rs` -- `C`/`Shift+C` keybind detection
- `src/ui/panels.rs` -- ComboBox in info panel, camera name + hint in
  grid footer

## Known issue

The grid footer hint text is now wider with the added `[C] camera` and
camera name label. When the window is narrow, the left-aligned info
(play icon, grid size, episode range, camera name) and the right-aligned
hint text overlap. The footer needs responsive layout -- either truncating
hints, using abbreviations, or hiding lower-priority items at small widths.
This will be addressed separately.

## Code quality notes

The implementation is 179 insertions across 5 files. Minimal and focused:

- `rebuild_video_paths()` eliminates duplication between load and switch
- `switch_camera()` / `cycle_camera()` cleanly separate "switch to index
  X" from "cycle by delta"
- The pending_camera_switch pattern is a one-field addition with a
  three-line consumer in update()
- No new structs or enums needed -- this is purely wiring existing
  infrastructure (VideoPlayer, GridView, EpisodeCache) to a new trigger

The changes are intentionally minimal because the hard infrastructure
(per-pane VideoPlayer, GridView rebuild, episode cache) already existed
from the grid view and slider scrubbing work. Camera switching is
conceptually "rebuild everything with different file paths" and the code
reflects that.

## Phase 2: Multi-camera single-episode view

### Goal

Show all (or selected) cameras for one episode simultaneously in an
auto-sized grid. Toggle with `M` keybind or View > Multi-Camera menu.

### Core changes

**GridMode enum** (`grid.rs`):
```rust
enum GridMode {
    MultiEpisode,  // one camera, multiple episodes (existing)
    MultiCamera,   // one episode, multiple cameras (new)
}
```

**GridPane gains identity**: Added `video_key: String` (full camera key,
stored for Feature 3) and `label: String` (display text for the pane
label -- "ep 001" in multi-episode, "wrist" in multi-camera).

**create_pane refactored**: Previously took an index into `GridDataset`
arrays. Now takes explicit (episode_index, video_key, label, video_path,
total_frames, seek_range, fps). This makes the two constructors
(`new` for multi-episode, `new_multi_camera` for multi-camera)
independent of each other -- each constructs panes with different
iteration patterns but shares the same low-level pane creation.

**new_multi_camera constructor**: Iterates `selected_cameras` (not
episodes), calls `ds.video_path(episode, camera)` per pane, auto-sizes
the grid:

| Cameras | Grid |
|---------|------|
| 1       | 1x1  |
| 2       | 2x1  |
| 3-4     | 2x2  |
| 5-6     | 2x3  |
| 7+      | ceil(sqrt(n)) x ceil(n/cols) |

### App state

Added `selected_cameras: Vec<bool>` -- one entry per video_key, all
initialized to `true` on dataset load. Persists across mode switches.

Added `pending_multi_camera_rebuild: bool` -- when camera checkboxes
change, the multi-camera grid rebuilds next frame (same deferred pattern
as `pending_camera_switch`).

### UI

**Camera checkboxes**: Appear in the info panel below the camera ComboBox,
only when in multi-camera mode. Each camera has a checkbox. At least one
must stay selected (the last checkbox can't be unchecked). Toggling a
checkbox triggers `pending_multi_camera_rebuild`.

**View menu**: Added "Multi-Camera [M]" / "Single Camera [M]" toggle,
only shown when dataset has >1 camera.

**Grid footer**: Multi-camera mode shows "ep XXX  N cameras" instead of
"Grid NxN" and episode range. Hint bar shows "[M] exit  [C] camera"
instead of "[G] exit  [+/-] resize  [←/→] page".

**Episode list**: Clicking an episode in multi-camera mode rebuilds the
grid for that episode (doesn't page like multi-episode mode).
Highlighting shows only the fixed episode, not a range.

### Mode interactions

| From | Press | Result |
|------|-------|--------|
| Single-video | M | Enter multi-camera grid |
| Multi-camera | M | Exit to single-video |
| Multi-camera | Escape | Exit to single-video |
| Multi-camera | G | Exit to single-video (same as any grid exit) |
| Multi-episode | M | No-op |
| Multi-camera | C | No-op (all cameras shown) |
| Multi-camera | +/- | No-op (auto-sized) |
| Multi-camera | ←/→ | No-op (single episode) |

### Bug fixes during review

1. **Duplicate trajectory pane**: In multi-camera mode, the info panel
   renders `show_trajectory_panel()` AND the standalone trajectory side
   panel also rendered. Fixed by skipping the standalone panel when
   `is_multi_camera`.

2. **Pane selection**: Click-to-select had no purpose in multi-camera mode.
   Disabled -- only fires in `MultiEpisode` mode.

### Files changed

- `src/grid.rs` -- GridMode enum, GridPane fields, refactored create_pane,
  new_multi_camera(), camera_grid_size(), mode-gated pane selection
- `src/app.rs` -- selected_cameras, pending_multi_camera_rebuild, layout
  changes for info panel visibility and trajectory panel gating
- `src/playback.rs` -- toggle_multi_camera(), GridMode import
- `src/ui/input.rs` -- M keybind, C disabled in multi-camera, +/-/arrows
  gated, escape enters video mode on exit
- `src/ui/panels.rs` -- camera checkboxes, View menu item, grid footer
  mode-aware rendering, episode list multi-camera navigation

## Phase 3: Multi-camera in grid — tiling + subgrid

### Evolution

Initial implementation used a separate `GridMode::EpisodeCamera` that
forced cols=cameras and rows=N. This overrode the user's grid layout
choice and felt disconnected from the episode grid.

Replaced with two approaches as a three-way M cycle within MultiEpisode:

### CameraDisplay enum

```rust
enum CameraDisplay {
    SingleCamera,  // one cam per pane (default)
    Tiled,         // each episode gets cam_count flat panes
    Subgrid,       // each episode cell renders cameras internally
}
```

M cycles through these in the multi-episode grid. The `GridMode` enum
was simplified to just `MultiEpisode` and `MultiCamera` (removed
`EpisodeCamera`).

### Tiled mode

`new_tiled()` snaps cols to `cam_count * groups_per_row` where
`groups_per_row = max(1, grid_cols / cam_count)`. Multiple episode
groups fit per row when the grid is wide. User's grid size preference
drives the layout — +/- resize and grid size picker work normally.

### Subgrid mode

`new_subgrid()` creates `grid_cols × grid_rows` episode cells, each
with `cam_count` sub-panes rendered inside it. The outer grid matches
the user's layout exactly. `show_subgrid()` subdivides each cell using
`camera_grid_size()` for the internal camera arrangement.

Key difference from tiled: episode count stays the same as single-camera
mode (e.g., 3×3 = 9 episodes with camera sub-frames), whereas tiled
reduces episode count to fit camera panes (e.g., 3×3 with 3 cameras
shows 3 episodes).

### Centralized grid construction

`enter_grid_with_camera_display()` dispatches to the right constructor
based on `camera_display`. Used by toggle_grid_view, toggle_multi_camera,
grid_resize, grid size picker, and pending camera checkbox rebuilds.
Eliminated duplicated construction logic.

### Key refactors

- `is_multi_camera()` → `is_camera_grid()`: returns true when
  `cam_count > 1` (covers both tiled and subgrid)
- `camera_tiling: bool` → `camera_display: CameraDisplay` enum
- `show()` split into `show_flat()` + `show_subgrid()` with shared
  `draw_pane_frame()` helper
- Grid size picker always visible (stores preference even in modes that
  auto-size)
- `navigate_page_tiled()` respects `subgrid` flag to reconstruct with
  the right constructor

### Bug fixes

1. Grid size picker resetting tiled mode to single-camera (called wrong
   rebuild path)
2. Subgrid showing wrong episode count (was reusing new_tiled layout
   math instead of grid_cols × grid_rows)

### Files changed

- `src/grid.rs` -- removed EpisodeCamera, added new_tiled/new_subgrid,
  show_flat/show_subgrid, draw_pane_frame, subgrid field
- `src/app.rs` -- CameraDisplay enum, is_camera_grid(), pending rebuild
- `src/playback.rs` -- enter_grid_with_camera_display(), three-way M
  cycle in toggle_multi_camera
- `src/ui/input.rs` -- simplified to MultiEpisode + MultiCamera match
- `src/ui/panels.rs` -- cycle-aware menu labels, always-visible grid
  picker, subgrid-aware episode list

## Phase 4: UI separation and dedicated controls

After testing Phase 3, the three-way M cycle felt awkward and the grid
size picker behavior was inconsistent across modes. Reorganized the
controls so each layout has its own entry point.

### Mental model

Three orthogonal display modes, each with its own control:

| Mode | Description | Entry |
|------|-------------|-------|
| SingleCamera | NxN episode grid, one camera per pane | Grid picker |
| Subgrid | NxN episode grid, all cameras inside each cell | M keybind |
| Tiled | Cols=cameras matrix layout | N keybind / row slider |

The grid picker (2D NxN) controls the **episode grid** dimensions and
always enters SingleCamera or Subgrid (whichever camera_display is set).
Tiled has its own row slider since cols are derived from cam_count.

### Keybind separation

- **M**: toggles SingleCamera ↔ Subgrid (and Single-video ↔ MultiCamera)
- **N**: toggles Tiled view on/off (no-op outside multi-episode grid)
- Both keybinds have menu equivalents

The previous three-way M cycle was confusing -- "Subgrid Cameras [M]"
when in Tiled mode made no sense. Two-way toggle is clearer.

### Matrix Rows slider

Always visible in the View menu when the dataset has multiple cameras.
Interacting with the slider auto-enters Tiled mode, so the user doesn't
need to press N first. The slider only adjusts row count -- cols are
derived from the selected camera count.

### Bug fixes during review

1. **Camera_display not reset on dataset load**: Loading a different
   dataset preserved stale tiled/subgrid state. Reset to SingleCamera +
   `grid_view = None` in load_dataset().

2. **Grid picker no-op in MultiCamera mode**: The picker click handler
   had a `if is_multi_camera { /* store only */ }` branch that swallowed
   clicks silently. Now it always rebuilds to the episode grid at the
   selected size.

3. **Subgrid showing wrong episode count**: Subgrid was reusing
   new_tiled() which computes episodes from tiled layout math
   (groups_per_row). Added new_subgrid() that creates grid_cols × grid_rows
   episode cells, each with cam_count panes.

### Refactoring

- Extracted `selected_video_keys()` helper replacing 3 duplicated
  filter/map patterns in new_multi_camera, new_tiled, new_subgrid

### Files changed (Phase 4)

- `src/playback.rs` -- toggle_tiled() method, simplified toggle_multi_camera
- `src/ui/input.rs` -- N keybind handling
- `src/ui/panels.rs` -- separate Tiled View [N] menu button, Matrix Rows
  slider always visible, grid picker bug fix
- `src/grid.rs` -- selected_video_keys() helper
- `src/app.rs` -- camera_display reset on dataset load

## Phase 5: Camera switch frame preservation

### Bug

Pressing C/Shift+C mid-playback reset the video to the episode start
(or the nearest keyframe). Switching cameras should preserve the frame
position so the user stays at the same point in the episode.

### Root cause

Two issues stacked:

1. **Player not re-seeked**: `switch_camera` called `enter_video_mode`
   which creates a fresh VideoPlayer starting at `episode_start_frame`.
   The preserved relative position was thrown away.

2. **Paused poll grabs keyframe frame**: Even after adding `player.seek()`
   to jump to the preserved position, the seek makes the decoder start
   from the nearest keyframe **before** the target (e.g. frame 80 when
   target is 100). The paused update loop polls once and receives that
   keyframe frame, overwriting `self.current_frame` via line:
   ```rust
   self.current_frame = player.current_frame;
   ```
   When the user pressed Play, playback continued from 80 instead of 100.

### Fix

Added `VideoPlayer::drain_to_frame(target, timeout_ms)` — a blocking
variant of poll that discards frames with `frame_index < target` and
returns the first frame at or after the target. Uses `recv_timeout`
with a 200ms bound so it can't hang the UI.

`switch_camera` now:
1. Captures `preserved_relative_frame` and `was_playing`
2. Rebuilds paths, caches, player
3. Calls `player.seek(abs_frame)` to start decoder near target
4. Calls `player.drain_to_frame(abs_frame, 200)` to synchronously
   skip intermediate keyframe frames and get the exact target texture
5. Restores `was_playing` state

Grid view has the same path via `GridView::seek_panes_to_relative`
which captures per-pane relative frames before rebuild and seeks each
new pane to its preserved position, draining intermediate frames.

### Why the drain is synchronous

`poll_next_frame` is non-blocking and only returns one frame at a
time. To guarantee the displayed frame matches the target, we need to
either:
1. Block briefly to drain frames (chosen)
2. Add state tracking across multiple update ticks to skip frames
   until target is reached

Option 1 is simpler and the decoder is fast enough that draining a few
intermediate frames completes in well under 200ms. The user sees no
visible rewind.

### Failed alternative

An earlier attempt modified `decode_all_frames_inner` to silently drop
frames where `actual_frame < seek_to_frame`. This worked but changed
core decoder behavior for all callers, which was too broad for a
camera-switch feature. Reverted in favor of the drain-at-call-site
approach that only affects camera switching.

### Files changed (Phase 5)

- `src/cache.rs` -- VideoPlayer::drain_to_frame()
- `src/playback.rs` -- switch_camera drains to exact frame after seek,
  per-pane relative frame preservation for grid path
- `src/grid.rs` -- seek_panes_to_relative() helper

## Phase 6: Composable mode transitions and state preservation polish

After more testing, rounded out the grid state management.

### Mode transition completeness

Previously G/M/N had asymmetric behavior from different starting states.
Normalized so each keybind is a pure dimension toggle:

| From \\ Key | G | M | N |
|------------|---|---|---|
| Single-video | multi-episode grid | multi-camera grid | grid + Tiled |
| Multi-episode grid | exit to single-video | + Subgrid | + Tiled |
| Multi-camera grid | + episode dim (Subgrid) | exit to single-video | + episode dim (Tiled) |
| Subgrid grid | exit | back to SingleCamera | switch to Tiled |
| Tiled grid | exit | switch to Subgrid | back to SingleCamera |

Both `single → M → G` and `single → G → M` now converge on Subgrid.
Both `single → M → N` and `single → N` now land in Tiled. N works from
single-video too (previously only worked inside the grid).

### Escape as panic button

Added `reset_to_initial_view()` method. Escape now:
- Exits any grid
- Switches to the default camera (wrist or index 0)
- Resets all camera checkboxes to checked
- Resets `camera_display` to SingleCamera
- Preserves the current episode / grid start

The idea: if the user is lost in a weird state combination, Escape
always returns to a known baseline without needing to remember which
keybinds to undo.

### State preservation across rebuilds

Every grid rebuild (camera switch, checkbox toggle, mode transition)
now preserves playback position, playing/paused state, and pane
selection. Extracted into a `PreservedGridState` struct with
`GridView::preserved_state()` capture and `restore_state()` apply
methods. Handles pane count mismatches (e.g., after checkbox toggle
changes camera count) by using the first captured frame as a
synchronized playback position for all new panes.

### Bug fixes

- **Frame jumped to keyframe on camera switch while paused**: `player.seek()`
  made the decoder start from the nearest keyframe, and the paused poll
  grabbed that first frame, overwriting `current_frame`. Fixed with
  `drain_to_frame` (Phase 5) plus proper position preservation.
- **Grid auto-played after camera switch**: `GridView::new` defaults to
  `playing: true`. Now capture `grid.playing` and restore after rebuild.
- **Pane selection lost on camera switch**: `selected_panes` is now
  cloned and restored via `PreservedGridState`.
- **Checkbox toggle reset video to beginning**: The pending rebuild
  handler now uses `preserved_state()` + `restore_state()`.
- **Exit grid returned to ep 0**: `toggle_grid_view` and Escape now set
  `current_episode = grid.start_episode` before returning to single-video.
- **Unchecking cameras to 1 broke the UI**: `is_camera_grid()` now
  checks `camera_display != SingleCamera` instead of `cam_count > 1`,
  so the checkboxes stay visible even with 1 camera selected.
- **Camera ComboBox bypassed multi-cam state**: ComboBox is hidden in
  multi-cam modes (where checkboxes are the authoritative control).
- **Grid picker reset tiled mode**: Picker now routes through
  `enter_grid_with_camera_display` so it respects `camera_display`.
- **Matrix Rows slider sent to episode 0 from single-video**: Fallback
  episode is now `self.current_episode`, not 0.

### Files changed (Phase 6)

- `src/grid.rs` -- PreservedGridState struct, preserved_state/restore_state methods
- `src/playback.rs` -- reset_to_initial_view, toggle_grid_view/toggle_multi_camera/toggle_tiled
  handle MultiCamera promotion, use PreservedGridState
- `src/app.rs` -- is_camera_grid uses camera_display, pending rebuild uses PreservedGridState
- `src/ui/input.rs` -- Escape calls reset_to_initial_view
- `src/ui/panels.rs` -- ComboBox hidden in multi-cam, "Tiled View Rows [N]" label, grid picker routing
