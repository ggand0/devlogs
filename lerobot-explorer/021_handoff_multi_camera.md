# Handoff: Multi-camera display (PR #4, merged 2026-04-15)

## Project overview

**tracelr** is a Rust/egui desktop tool for exploring LeRobot robotics datasets.
It displays video episodes with frame scrubbing, EE trajectory visualization
via forward kinematics from URDF files, and a grid view for comparing multiple
episodes. The codebase is ~5k lines of Rust across ~15 source files.

Key run command: `RUST_LOG=tracelr=debug cargo run --profile opt-dev -- /path/to/dataset/`

## What was shipped in this PR

Multi-camera display support. LeRobot datasets have 2-6 cameras (wrist, top,
side, etc.) but the app previously only showed one camera at a time.

### Features added

**Camera cycling** (`C` / `Shift+C`):
- ComboBox dropdown in the Episode Info panel (single-video mode) or
  Cameras panel (grid mode) to select which camera to display
- Keyboard shortcuts cycle forward/backward through cameras
- Frame position, playing state, and pane selection are preserved across
  camera switches via `VideoPlayer::drain_to_frame()` which blocks briefly
  to skip intermediate keyframe frames

**Multi-camera single-episode view** (`M` from single video):
- Auto-sized grid showing all cameras for one episode (2 cams = 2x1,
  3-4 = 2x2, 5-6 = 2x3)
- Camera checkboxes in the Cameras panel to enable/disable individual cameras
- At least one camera must remain checked

**Multi-camera in grid view** (two layouts):
- **Subgrid** (`M` from grid): each episode cell renders all cameras as
  mini-frames inside it, preserving the user's NxN grid layout
- **Tiled** (`N` from grid): each camera gets its own flat pane, cols snap
  to a multiple of camera count. Has its own "Tiled View Rows" slider in
  the View menu.

**Composable keybinds**:
- `G` toggles the episode dimension
- `M` toggles the camera dimension
- `N` toggles the Tiled layout
- All three compose: `M` then `G` = Subgrid, `M` then `N` = Tiled, etc.
- `Escape` is a panic button that resets to the initial single-video view
  (default camera, all checkboxes re-checked, episode preserved)

### Key architecture

**Enums**:
- `GridMode`: `MultiEpisode` | `MultiCamera` (2 variants, down from 3;
  the old `EpisodeCamera` matrix mode was replaced by smart tiling within
  `MultiEpisode`)
- `CameraDisplay` on `App`: `SingleCamera` | `Tiled` | `Subgrid`

**GridView constructors**:
- `new()` for single-cam multi-episode
- `new_multi_camera()` for M from single video (auto-sized by camera count)
- `new_tiled()` for tiled layout (cols = cam_count * groups_per_row)
- `new_subgrid()` for subgrid layout (cols = grid_cols, cam_count sub-panes
  per cell)

**State preservation**:
- `PreservedGridState` struct captures per-pane relative frames, playing
  state, and selected_panes before any grid rebuild
- `GridView::preserved_state()` / `restore_state()` methods
- Used in `switch_camera`, `pending_multi_camera_rebuild`, and checkbox toggles
- `VideoPlayer::drain_to_frame(target, timeout_ms)` blocks briefly after
  `seek()` to skip decoder keyframe frames and return the exact target frame

**Rendering**:
- `show()` dispatches to `show_flat()` or `show_subgrid()` based on the
  `subgrid` flag on `GridView`
- `draw_pane_frame()` is shared between both renderers
- `show_camera_combobox()` helper shared between `show_info_panel` (single
  video) and `show_cameras_panel` (grid mode)

**UI panels**:
- Single-video mode: right panel is "Episode Info" (with camera ComboBox,
  task, trajectory)
- Any grid mode: right panel is "Cameras" (camera ComboBox or checkboxes,
  trajectory). Episode Info section is hidden in grid modes because it
  showed stale `current_episode` data that didn't match the visible grid.
- `grid_dataset!()` macro builds `GridDataset` from `App` fields using
  direct field access (not a method) so the borrow checker can track
  individual borrows without conflicting with `&mut self.grid_view`

**Helpers**:
- `camera_display_name()` strips `observation.images.` prefix from video keys
- `selected_video_keys()` filters video keys by the `selected_cameras` bitmask
- `is_camera_grid()` returns true when the grid shows multiple cameras
  (checks `camera_display != SingleCamera`)
- `enter_grid_with_camera_display()` dispatches to the right constructor
  based on `camera_display`
- `reset_to_initial_view()` is the Escape panic button implementation

## Key files and what they do

| File | Role |
|------|------|
| `src/grid.rs` | GridView, GridPane, GridMode, CameraDisplay constructors, PreservedGridState, show_flat/show_subgrid, camera_grid_size |
| `src/app.rs` | App state, CameraDisplay enum, is_camera_grid(), rebuild_video_paths(), pending action handlers, layout routing |
| `src/playback.rs` | toggle_grid_view/toggle_multi_camera/toggle_tiled, switch_camera/cycle_camera, enter_grid_with_camera_display, reset_to_initial_view, grid_resize |
| `src/ui/panels.rs` | show_info_panel, show_cameras_panel, show_camera_combobox, show_grid_footer, View menu (grid picker, Tiled View Rows slider, M/N buttons), episode list highlighting |
| `src/ui/input.rs` | Keyboard handler (G/M/N/C/Escape), handle_keyboard_grid, handle_keyboard_grid_tiled |
| `src/cache.rs` | VideoPlayer, drain_to_frame(), DecodeLruCache |
| `src/dataset.rs` | camera_display_name(), selected_video_keys() |
| `src/main.rs` | grid_dataset!() macro |
| `src/video.rs` | decode_all_frames_sync, scrub_worker |

## Known issues and future work

**Known limitation**: the grid footer hint text overflows at narrow window
widths, causing the right-aligned hints to overlap the left-aligned status.
Needs responsive layout.

**README gap**: Windows URDF setup instructions are missing. Linux and
macOS are documented under "Setting up a URDF."

**Untested in PR**: Dataset switching (loading a new dataset while in
Subgrid/Tiled) and platform-specific items (Retina rendering, trackpad,
cmd+click). These are covered in the test plan in the PR description but
were not verified.

**Future ideas discussed but not implemented**:
- 3D camera view: camera feeds as textured planes in the existing
  trajectory 3D scene, positioned using camera extrinsics from URDF.
  The OrbitCamera and egui 3D rendering infrastructure already exist from
  the trajectory work.
- The subgrid approach could be extended with per-camera labels inside
  each cell (currently only the episode label is shown at the bottom).

## User preferences (from CLAUDE.md and feedback)

- Present tense verb format for commit titles ("Add...", "Fix...", "Remove...")
- No Claude Code footer in commits (no Co-Authored-By, no Generated with)
- No em dashes anywhere (commit messages, devlogs, PR drafts, source comments)
- Do not `git add -A`; add individual files
- Do not commit devlog markdowns
- Do not git push/checkout/revert without permission
- `docs/plans/` is gitignored; plans stay local
- The user wants fundamental fixes, not superficial hacks
- When the user asks "should we do X?" give a direct answer, not hedging

## Devlog

Full implementation log (Phases 1-6) is at
`/data/devlogs/lerobot-explorer/020_camera_cycling.md`. It covers the
iterative design evolution: camera cycling, multi-camera grid, the
EpisodeCamera matrix mode (added then replaced), smart tiling, subgrid,
composable mode transitions, Escape reset, and all the state preservation
bug fixes.
