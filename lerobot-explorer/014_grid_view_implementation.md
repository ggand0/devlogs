# Grid view for multi-episode playback

Date: 2026-03-29

## Motivation

The single-video view makes it hard to compare episodes or spot outliers
(failed grasps, awkward motions) across a dataset. A grid view lets you
play multiple episodes side-by-side to visually scan for problems.

## Architecture

### GridView struct (`src/grid.rs`)

`GridView` manages N `GridPane` instances, each wrapping a `VideoPlayer`:

```
GridView
├── panes: Vec<GridPane>      (one per visible episode)
│   ├── episode_index         (which episode this pane shows)
│   ├── player: VideoPlayer   (independent decode thread + channel)
│   ├── current_texture       (latest decoded frame as egui TextureHandle)
│   ├── current_frame         (absolute frame position)
│   ├── episode_start_frame   (for v3.0 concatenated videos)
│   └── total_frames          (episode length)
├── cols, rows                (grid dimensions, e.g. 2x2 or 3x2)
├── start_episode             (first episode in the grid, top-left pane)
├── playing                   (play/pause state for all panes)
├── fps                       (dataset fps, shared across panes)
└── selected_pane             (click-to-select highlighting)
```

Each `VideoPlayer` spawns its own decode thread with a bounded channel
(`sync_channel(30)`). The main thread polls all players each frame via
`tick()`, which respects the 1/fps frame duration.

`GridDataset` bundles the dataset references (video_paths, seek_ranges,
episodes, fps) to avoid passing 4+ arguments through every method. All
pane rebuild operations (navigate, jump, resize) go through a shared
`rebuild()` helper.

### Concurrency model

- N decode threads run concurrently (one per pane)
- Each thread decodes its own video file independently
- Main thread polls `try_recv()` on all channels each frame
- `Drop` on `VideoPlayer` sets an `AtomicBool` cancel flag, stopping the thread
- Page navigation drops all panes (cancels threads) and creates new ones

At 4x4 (16 panes), this means 16 concurrent decode threads. SW decode
throughput at 480p is ~680fps (H.264) / ~312fps (AV1), so even splitting
across 16 threads leaves ~42fps/~19fps per stream — enough for 30fps
playback on H.264, tight on AV1.

### Rendering

Grid rendering uses `ui.allocate_rect()` for each pane cell and paints
directly via `ui.painter()`:

1. Divide available central panel space into `cols x rows` cells with 4px spacing
2. For each pane: fill background, draw video frame (aspect-fit), draw episode
   label at bottom, draw selection border if selected
3. Video frames are drawn with `painter.image()` using the TextureHandle from
   the latest polled frame

No egui layout system is used inside the grid — it's all manual rect math.
This avoids layout overhead and gives precise control over pane sizing.

### Integration with App

Grid mode is toggled via `App.grid_view: Option<GridView>`. When `Some`:
- The update loop calls `grid.tick(ctx)` instead of single-player tick
- The episode list remains visible with all grid episodes highlighted
- Right panel (info/annotation) is hidden to maximize grid space
- Bottom bar shows grid-specific footer (grid size, episode range, controls)
- Central panel calls `show_grid_display()` instead of `show_frame_display()`
- Keyboard routes to grid-specific handlers (page nav, resize)

When `None`, the app behaves exactly as before (single-video mode).

### Episode list integration

The episode list stays visible in grid mode. All episodes currently shown
in the grid are highlighted (using `grid.start_episode..start_episode + pane_count`).
Clicking an episode in the list jumps the grid to start at that episode.

Auto-scrolling uses `ui.scroll_to_rect()` with the union rect of all
selected episode rows. The scroll fires once per navigation event via a
`scroll_to_selected` flag, so it doesn't fight manual scrolling.

### Grid size picker

The View menu contains an interactive 4x4 mini-grid widget (similar to
Google Docs table insertion). Hovering highlights from (1,1) to the cursor
position, clicking applies that size. Supports non-square layouts (e.g.
3x2, 1x4). Selecting a size from single view enters grid mode directly.

### Controls

| Key | Action |
|-----|--------|
| G | Toggle grid/single view |
| +/- | Resize grid (1x1 ↔ 2x2 ↔ 3x3 ↔ 4x4, square only) |
| Left/Right or A/D | Page through episodes (shifts by cols×rows) |
| Space | Play/pause all panes |
| Escape | Exit grid, return to single view |
| Click pane | Select pane (highlight border) |

View menu → Grid View / Single View toggle + grid size picker for
arbitrary dimensions.

### Dataset reload (drag-and-drop)

When a new dataset is dropped while grid view is active, the old grid
is torn down, the new dataset loads, and grid mode re-enters automatically
with the same grid dimensions.

### Renderer

Switched from eframe's default glow (OpenGL/GLX) renderer to wgpu (Vulkan).
The glow backend was failing with GLX `BadValue` errors on X11. eframe's
`wgpu` feature was not enabled by default — now explicitly set in Cargo.toml
with `x11` and `wayland` platform features.

## Performance

### Measured FPS (H.264 640x480, desktop PC)

| Grid | Panes | App FPS |
|------|-------|---------|
| 2x2  | 4     | ~72     |
| 3x3  | 9     | ~60     |
| 4x4  | 16    | ~45     |
| 5x5  | 25    | ~30     |
| 6x6  | 36    | ~22     |

### Raw decode throughput (single-file benchmark)

- H.264 480p SW: ~681 fps
- AV1 480p SW: ~312 fps

At 6x6, 681/36 ≈ 19 fps/stream from decode alone — but app FPS is 22,
meaning the bottleneck is **not** decode throughput. The main thread is
the bottleneck.

### Pipeline per frame

```
Decode threads (N concurrent):
  1. ffmpeg decode (YUV)              — CPU
  2. swscale YUV→RGBA                 — CPU, new Context per frame
  3. frame_to_color_image()           — per-pixel loop (307K iterations)
  4. send ColorImage via channel      — 1.2MB per frame

Main thread (single, per tick):
  5. try_recv() × N panes             — poll channels
  6. ctx.load_texture() × N panes     — create NEW texture + 1.2MB GPU upload
  7. egui render                      — draw N textures
```

### Bottlenecks identified

1. **ctx.load_texture() per frame per pane** — creates a new GPU texture
   handle every frame. At 6x6, that's 36 texture creations + uploads per
   tick on the main thread. Old textures get freed but the churn is expensive.
   Fix: reuse texture handles via `texture.set()` instead of creating new ones.

2. **frame_to_color_image() per-pixel loop** (video.rs:155) — pushes 307K
   Color32 values one at a time. `Color32` is `[u8; 4]`, same memory layout
   as RGBA. When stride == width*4 (common), this could be a zero-cost
   reinterpret (`from_raw_parts`) instead of a loop. Even when stride differs,
   row-wise memcpy is much faster than pixel-by-pixel.

3. **swscale Context created per frame** (video.rs:139) — `scaling::Context::get()`
   allocates and initializes a new scaler for every frame, even though
   format/dimensions don't change within an episode. Should be created once
   per decode thread and reused.

### Optimization priorities (not done in this branch)

| Fix | Expected impact | Complexity |
|-----|-----------------|------------|
| Reuse texture handles (`texture.set()`) | High — eliminates N texture allocs/frame on main thread | Low |
| Zero-copy RGBA→Color32 conversion | Medium — eliminates 307K×N iterations/frame on decode threads | Low |
| Reuse swscale Context per thread | Medium — eliminates scaler init per frame | Low |
| Lower resolution for grid panes | High — 4x fewer pixels at 320x240 | Medium |

All three code fixes are low complexity and could 2-3x the grid FPS.

## Commits on this branch

1. Add grid view for simultaneous multi-episode playback
2. Add grid size picker and show episode list in grid mode
3. Fix dataset drag-and-drop while grid view is active
4. Switch from glow (OpenGL) to wgpu (Vulkan) renderer
5. Show grid size picker in View menu regardless of current mode
6. Fix grid size picker not activating when clicked size matches default
7. Auto-scroll episode list to follow grid view navigation
8. Fix auto-scroll to show full grid range using scroll_to_rect
9. Fix clippy warnings, refactor GridView to use GridDataset + rebuild()
10. Increase max grid size from 4x4 to 6x6
11. Extract magic numbers in grid rendering to named constants

## Future improvements

- Double-click pane to open that episode in single view
- Per-pane play/pause
- Episode metadata overlay on grid panes
- Episode filtering (show only unannotated, or episodes matching criteria)
- Thumbnail grid mode (static frames, no playback) for faster scanning
- Performance optimizations (texture reuse, zero-copy RGBA, swscale caching)
