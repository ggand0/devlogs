# 022: Shareable episode export (2026-10-16)

Branch: `feat/episode-export`. Plan: `docs/plans/010_episode_export.md` (all four phases).

## What the user gets

| Action | Trigger | Output |
|---|---|---|
| Copy frame | `Cmd/Ctrl+C` in the viewer, `File > Copy Frame`, or the Export window | composite of the frame on screen on the clipboard |
| Save frame | `File > Save Frame as PNG...` | same composite as a png |
| Export clip | `File > Export Clip...` -> Export window -> `Save Clip...` | one h264 yuv420p mp4 of the whole episode |
| Export report | `File > Export Report...` -> Export window -> `Export Report...` | `<dataset>_ep<NNNN>/` folder (see below) |

Scope is the current episode only. Everything runs on one background thread per
job with a progress bar + Cancel at the bottom of the window and a 4 s message at
the bottom right when it finishes.

## Output files and job safety

- Clip and PNG exports write to a hidden sibling (`.<name>.partial.<ext>`, the
  extension kept so ffmpeg and the image crate pick the format) and rename it
  onto the target only on success (`write_replacing`). Earlier version wrote to
  the target directly and deleted it on error or cancel, which lost an existing
  file when re-exporting over it and cancelling.
- The export thread runs `run` under `catch_unwind`; a panic becomes
  `ExportEvent::Failed("export crashed: ...")`. `poll_export_job` also treats a
  disconnected channel as a failure, so the progress bar never hangs.

## Pipeline

```
EpisodeFrames (per camera) -> Compositor (tiles + trajectory) -> sink
                                                                   clipboard image
                                                                   png
                                                                   Mp4Writer
                                                                   report folder (N+1 Mp4Writers)
```

Modules:

```
src/export/
  mod.rs          ExportRequest / Layout / FrameRange / Sink / ExportEvent / ExportJob,
                  spawn(), run(), Compositor, clip + report writers
  source.rs       EpisodeFrames: sequential per-camera reader with v3.0 trim
  compose.rs      Canvas, plan() (tile layout), auto_rows(), blit()
  encoder.rs      Mp4Writer: RGBA -> yuv420p -> libx264 (mpeg4 fallback) mp4
  traj_render.rs  TrajectoryRenderer: offscreen egui + wgpu render of the trajectory panel
  report.rs       ReportMeta, summary.md / metadata.json, RFC 3339 timestamp
src/ui/export_dialog.rs  ExportDialogState, Export window, menu items, job polling,
                         clipboard, progress bar, toast
```

### EpisodeFrames (`source.rs`)

- `open(path, seek_range, max_frames)`: seeks to the keyframe at or before
  `from_s`, then drops frames whose PTS-derived index is `< round(from_s * fps)`
  and stops at the first index `>= round(to_s * fps)`. `max_frames`
  (episode length) is a safety bound.
- `next()` converts to `ColorImage`; `skip(n)` decodes without the RGBA
  conversion (used to reach a single frame exactly, no inexact keyframe seek).
- `next_or_last()` returns the last frame again for a camera that ended early, so
  a single-frame export never fails on a camera that is a frame or two short.
- Implementation detail: decoded frames go into `scratch` and are swapped into
  `current` only when they belong to the episode, so `current` always holds the
  last good frame.
- `video::decode_episode_start_frame` now uses `EpisodeFrames` (refactor, not a
  copy). `decode_first_frame` (dead code) was removed. `ensure_ffmpeg_init` and
  `frame_to_color_image` became `pub(crate)`.

### Compositor and layout (`mod.rs`, `compose.rs`)

- A camera that ends early keeps showing its last frame in the composite. The
  clip stops once every camera has ended.
- `plan(sizes, with_traj, scale, rows)` takes each camera's native size in tile order:
  - Every camera keeps its own aspect ratio. Tiles in a row share one height,
    each tile's width is `height * aspect`, and every row is scaled to the width
    of the first row (whose height is the first camera's height times `scale`).
    Frames sit edge to edge with no bars and nothing cropped; cameras of the
    same size give a plain grid. The last cell of a row absorbs the rounding.
  - First version sized every tile from the first camera and letterboxed the
    others, which put black bars between frames on mixed rigs (nova: 16:9
    `head` 640x360 + 16:10 hand cameras 960x600). Now that rig exports as
    896x180 at half scale with no bars.
  - The trajectory panel is one more tile, always in the bottom-right cell.
    Cameras fill the other cells row-major, so empty cells sit just before it.
    The panel and empty cells use the first camera's aspect ratio. Empty cells
    are returned in `Plan.empty`.
  - `cols = ceil(cells / rows)`, then rows are recomputed from cols so an
    unfillable request (3 rows for 4 cells) collapses instead of leaving empty rows.
  - All dimensions are rounded down to even (yuv420p).
- `auto_rows` (rows = Auto):

  | Cameras | No trajectory | With trajectory |
  |---|---|---|
  | 1 | 1x1 | 1 col x 2 rows (camera on top) |
  | 2 | 1x2 | 2x2 (bottom-left empty) |
  | 3 | 1x3 | 2x2 |
  | 4 | 2x2 | 2x3 (one empty) |
  | 5 | 2x3 (one empty) | 2x3 |
  | 6 | 2x3 | 2x4 (one empty) |

  Rule: one row for up to 3 cameras without the trajectory, otherwise two rows
  up to 8 cells, beyond that a near-square grid.

### Mp4Writer (`encoder.rs`)

- `libx264` with `preset=veryfast crf=20`; falls back to built-in `mpeg4` at
  4 Mb/s with a warning, and the finish message says which codec was used.
- RGBA -> YUV420P via swscale, tagged BT.601 (`SMPTE170M`) limited range so
  players interpret colours the way swscale produced them.
- Frame pts are frame indices in `1/fps`; packets are rescaled to the stream time
  base after `write_header`.
- **Last-frame bug found during testing:** packets came out of the encoder with
  duration 0 (frames carry none). The mp4 muxer ends the edit list at the last
  pts + duration, so players dropped the last frame (v3.0 episode 5 decoded 399
  of 400 frames). The v2.1 test file only looked right because of millisecond
  rounding in the edit list. Fix: set `duration = 1` (one frame) on packets that
  have none, before `rescale_ts`.
- ffmpeg log level is set to Warning in `ensure_ffmpeg_init` so x264's statistics
  do not flood the terminal.

### TrajectoryRenderer (`traj_render.rs`)

- Own `egui::Context` and `egui_wgpu::Renderer` on the export thread, using
  clones of eframe's `wgpu::Device` / `Queue` (`App.gpu`, taken from
  `cc.wgpu_render_state` in `App::new`). The eframe renderer is not shared.
- Calls the existing `trajectory_view::show_trajectory_overlay` (no reimplemented
  drawing) with the user's `OrbitCamera` (now `#[derive(Clone)]`) and accent colour.
- Renders into an `Rgba8UnormSrgb` texture of any `width x height` (the panel fills
  a full camera tile), copies to a readback buffer with rows padded to 256 bytes,
  maps it and strips the padding. sRGB target means the bytes are already gamma
  encoded.
- `pixels_per_point = max(1, short_side / 320)`, so text and line widths look like
  the on-screen panel at any export resolution.
- The overlay's auto-zoom is written back into the renderer's camera on the first
  frame and persists.
- Report `trajectory.png` uses a separate 720x720 renderer at the last frame
  (whole path drawn).

### Report (`report.rs`, `write_report`)

```
<picked>/<dataset>_ep<NNNN>/
  clips/<camera>.mp4   raw per-camera clip, native size (rounded to even)
  composite.mp4        same as Export Clip with the window's options
  thumbnail.png        composite at frame 0
  trajectory.png       720x720 at the last frame (only when a trajectory exists)
  summary.md
  metadata.json        ReportMeta via serde_json::to_string_pretty
```

- Videos are decoded once; the composite and all per-camera writers are fed from
  the same loop.
- `trajectory.png` is written whenever a trajectory is available, even if the
  composite does not include the panel.
- An existing non-empty target folder is an error (no overwrite). On error or
  cancel, the folder is removed, or emptied if it existed (empty) before.
- `exported_at` is RFC 3339 UTC computed from `SystemTime` (chrono is a build-dep
  only), using Howard Hinnant's days-to-civil algorithm.

### UI (`export_dialog.rs`, `input.rs`, `panels.rs`, `app.rs`)

- Export window:
  - camera list in tile order with checkboxes (the last checked one is locked)
    and up/down buttons for order. The triangles are painted shapes
    (`arrow_button`): egui's default fonts have no U+25B2 / U+25BC glyphs and
    rendered them as empty boxes.
  - "Include trajectory panel" (disabled with a tooltip without URDF or GPU)
  - Rows: Auto / 1 / 2 / 3 (values larger than the cell count are disabled) and
    Scale: Full / Half, both as a pill-shaped segmented control (`segmented`):
    rounded track, selected option filled with the accent colour, disabled
    options dimmed. `selectable_value` looked like plain text.
  - live layout preview: boxes from the same `compose::plan` the export uses,
    trajectory cell in the accent colour, plus the output resolution.
    Tiles are drawn edge to edge with no borders, like the export: the canvas is
    filled with the tile colour first and only empty cells and the trajectory
    cell are painted over, so adjacent tiles show no anti-aliased seams.
    Tile sizes come from `info.json` (`DatasetInfo.video_sizes`, parsed from each
    video feature's `shape` + `names`), so no video is decoded for it.
  - buttons: Copy Frame, Save Clip..., Export Report...
  - options are kept for the session, not persisted.
- What the export targets (`view_target`):
  - single view: current episode, `current_frame - episode_start_frame`
    (0 outside video mode), the current camera only
  - multi-camera grid: the fixed episode, its pane frame, the selected cameras
  - episode grid: the lowest selected pane's episode (else the first pane), that
    pane's frame
- Cmd/Ctrl+C and `File > Copy Frame` use the cameras on screen, ordered by the
  Export window's order, with its rows setting, at full scale. The trajectory is
  included when the panel is visible.
- Cmd/Ctrl+C is detected as `egui::Event::Copy`. `C` / `Shift+C` camera cycling now
  ignores the key when Cmd/Ctrl is held. A focused text field keeps normal copy
  because `handle_keyboard` already returns early when a widget has focus.
- Clipboard: `arboard` on the UI thread. The `Clipboard` is kept in `App` so the
  image stays available on X11/Wayland.
- Menu items are disabled while a job runs; only one job at a time.
- `App::ensure_trajectory_cached(ep)` was extracted from the trajectory panel and
  is shared by the panel and the export.

## Dependencies

`arboard 3.6`, `egui-wgpu 0.31`, `wgpu 24` (both already transitive via eframe;
`cargo tree -i wgpu` still shows one version), dev-dependency `pollster 0.4`.

## Tests

- `compose`: single camera, one-row rule, trajectory bottom-right for 1-4 cameras,
  explicit rows (including collapse and clamp), `auto_rows` for 2-6 cameras,
  even dimensions for odd input, mixed aspect ratios with no bars (nova rig),
  justified rows tiling the canvas exactly, empty cells, blit filling its tile.
- `encoder`: 12-frame red/blue encode -> decode round trip through `EpisodeFrames`.
- `report`: RFC 3339 dates, summary with and without labels/annotation.
- `dataset`: video size from a named / unnamed `shape`.
- `traj_render` (ignored, needs a GPU): `cargo test -- --ignored traj_render`.

## Verified on real data

Headless runs of `export::run` (temporary test, not committed):

- v2.1 `so101_pick_place_10cm_v3` ep 0: 1-camera and 2-camera + trajectory clips,
  300/300 frames (`ffprobe -count_frames`).
- v3.0 `practice-pick-place-single-cell-10ep` ep 5: 400/400 frames in the
  composite and both per-camera clips. Frame 0 and the last frame match file
  frames 2249 and 2648 pixel for pixel, and ep 6 frame 0 matches 2649, so there is
  no leak across episodes.
- Report folder contents and the second-export error.
- Composite images for 1/2/3 cameras with the trajectory and 3 without, checked
  visually.
- Mixed aspect ratios: nova ep 0 (head 16:9 + two 16:10 hand cameras, half
  scale) gives 896x180 with no bars, 1236/1236 frames; practice ep 5 (4:3 +
  16:9 cameras + trajectory) gives two justified rows with no gaps.

Not yet verified by hand in the GUI: the menu, Export window, preview, Cmd+C
paste, cancel, and the mpeg4 fallback (this machine has libx264).
