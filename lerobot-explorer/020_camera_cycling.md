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
