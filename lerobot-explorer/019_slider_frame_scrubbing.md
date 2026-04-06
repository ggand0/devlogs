# Devlog 019 -- Slider frame scrubbing

## Goal

Dragging the frame slider should update the RGB video frame in real time,
not just the trajectory playhead. This lets you trace robot movements by
correlating RGB frames with EE trajectories interactively.

## Problem

The existing slider drag only updated `current_frame` (which the trajectory
visualization reads directly), but deferred the actual video decode to drag
release via `player.seek()`. The video frame stayed frozen during the entire
drag.

## First attempt: throttled seek() during drag

Called `player.seek()` at ~15 Hz during drag, with `poll_next_frame()` in the
update loop picking up decoded frames.

**Result:** Flickering "micro movements" from the same region of video.

**Root cause:** `seek()` spawns a sequential decoder thread that starts from
the nearest H.264/H.265 keyframe and decodes forward continuously. Keyframes
are typically 1-2 seconds apart (30-60 frames). At 15 Hz scrub rate, each
new `seek()` call:

1. Cancels the previous decoder thread
2. Spawns a new one that seeks to the same nearby keyframe
3. Starts decoding from that keyframe forward (frame 96, 97, 98...)

`poll_next_frame()` picks up whatever frame the thread managed to decode
before cancellation -- always one of the early frames near the keyframe, never
the actual target. So the display shows frames 96, 97, 96, 97, 96 flickering.

## Working implementation: dedicated scrub decoder

Separated scrubbing from sequential playback with a purpose-built decode path.

### video.rs -- `decode_frame_at()`

New function that seeks to the nearest keyframe and silently decodes forward,
discarding every frame until reaching `>= target_frame`. Returns only that
single frame. Accepts a cancel flag for early termination.

Key differences from `decode_all_frames_sync()`:
- Returns one frame, not a stream
- Discards all intermediate frames (no channel sends, no flickering)
- Checks cancel flag between frames for responsiveness

### cache.rs -- VideoPlayer scrub methods

Three new methods on `VideoPlayer`, separate from the main playback channel:

- `scrub_to(frame)` -- cancels any in-flight scrub, spawns a thread calling
  `decode_frame_at()`, sends result through a dedicated `mpsc::channel`
- `poll_scrub_frame()` -- non-blocking receive on the scrub channel, returns
  `Option<ColorImage>`
- `cancel_scrub()` -- sets cancel flag, drops channel

The scrub channel (`scrub_rx`, `scrub_cancel`) is independent of the main
playback channel (`frame_rx`, `cancel`). Scrubbing does not interfere with
the sequential decoder state.

### ui/panels.rs -- throttled scrub in slider drag

During drag, when `current_frame` changes and >= 67ms have elapsed since
the last scrub (15 Hz throttle), calls `player.scrub_to(current_frame)`.

On drag release: cancels scrub, calls `player.seek()` to position the
sequential decoder for playback.

### app.rs -- scrub frame polling

Added a poll path in `update()` for when `frame_slider_dragging` is true:
polls `player.poll_scrub_frame()` and uploads the `ColorImage` as a texture.
Importantly, does NOT overwrite `current_frame` from the decoder's PTS
estimate -- the slider-set value stays authoritative so the trajectory
playhead matches the slider position exactly.

## Data flow

```
Drag slider to frame 150
  |
  v
show_frame_slider(): current_frame = 150, throttle check passes
  |
  v
player.scrub_to(150): cancel old scrub, spawn thread
  |
  v
Thread: decode_frame_at(path, 150, fps, cancel)
  |-- ffmpeg seek to ~150/fps seconds (lands on keyframe, e.g. frame 128)
  |-- decode 128 (discard), 129 (discard), ..., 149 (discard)
  |-- decode 150 -> frame_to_color_image() -> send through channel
  |
  v
update(): poll_scrub_frame() receives ColorImage
  |-- load_texture() -> current_texture updated
  |-- trajectory already updated (reads current_frame directly)
  |
  v
Both RGB frame and trajectory show frame 150
```

## Iteration 2: persistent scrub worker

The initial `decode_frame_at()` approach worked but was slow: each scrub
request spawned a new thread, opened the video file, initialized the codec,
seeked, and decoded forward. At 15 Hz that's 15 file opens + codec inits
per second of pure overhead.

### Problem with per-scrub threads

| Operation             | Cost per scrub |
|-----------------------|----------------|
| Thread spawn          | ~50-100 us     |
| File open + demux     | ~2-5 ms        |
| Codec context init    | ~1-3 ms        |
| Seek to keyframe      | ~1-2 ms        |
| Decode forward N frames | ~2-5 ms/frame |

At 15 Hz, file open + codec init alone burns 45-120ms/sec. The actual
decode work (reaching the target frame) is a fraction of total time.

### Persistent worker design

Replaced the per-scrub pattern with `scrub_worker()` in video.rs: a single
long-lived thread that holds an open ffmpeg `Input` + `Decoder` for the
lifetime of the `VideoPlayer`.

**Key behaviors:**

1. **Blocking receive loop:** Worker blocks on `request_rx.recv()` when
   idle. Zero CPU usage between drags.

2. **Drain-to-latest:** On wakeup, drains all queued requests and only
   processes the most recent target. Stale intermediate positions are
   discarded.

3. **Forward scrub fast path:** If target > last_decoded_pos and within
   60 frames, the worker just calls `ictx.packets()` to continue decoding
   forward from where it left off. No seek, no flush. This is the common
   case when dragging rightward -- cost is just the few frames of decode
   between positions.

4. **Seek on direction change:** Backward scrubs or jumps > 60 frames
   trigger `ictx.seek()` + `decoder.flush()`. Cost is one seek + decode
   forward from keyframe.

5. **Lazy creation:** Worker thread is spawned on first `scrub_to()` call,
   not on VideoPlayer construction. Grid panes that never scrub don't
   pay for an idle thread.

6. **Clean shutdown:** Dropping the `scrub_tx` sender closes the channel,
   causing `recv()` to return `Err`, and the worker exits.

### VideoPlayer scrub API

```
scrub_to(frame)       -- drain stale results, lazily spawn worker, send target
poll_scrub_frame()    -- try_recv on result channel
cancel_scrub()        -- drain pending results (worker stays alive for reuse)
```

### Why forward scrub skips seeking

ffmpeg's `Input` maintains an internal read position. When we break out of
`ictx.packets()` after finding the target frame, the next `ictx.packets()`
call resumes from where we left off. No seek needed. The decoder also
retains its state -- no flush, no lost context, just keep feeding packets.

For a 30fps video scrubbing at 15 Hz, the target advances ~2 frames
between requests. Decoding 2 frames: ~4-10ms. Compared to seek + decode
from keyframe: ~30-80ms. Roughly 5-10x faster for the common case.

## Performance characteristics

- **Forward scrub (common):** ~2-10ms per update (decode a few frames)
- **Backward / large jump:** ~30-80ms (seek + decode from keyframe)
- **Worker idle:** blocks on recv(), zero CPU
- Scrub latency depends on keyframe distance for backward scrubs: if
  keyframes are every 30 frames, worst case is decoding ~30 frames to
  reach the target (~60-150ms for AV1 at 480x640)
- 15 Hz throttle means at most ~15 requests/second during drag, all
  handled by the single persistent worker thread

## Feature 2: instant playback on slider release

One-liner: set `self.playing = true` and `self.last_frame_time = None` on
drag release, so playback resumes immediately from the new position. Without
this the user had to press Space after every scrub.

## Feature 3: grid view frame slider

### Design decisions

- **Global slider, not per-pane.** A single slider below the grid controls
  all panes simultaneously. Per-pane sliders would be unusably small in a
  3x3+ grid.
- **Relative frame position.** The slider maps to 0..max_episode_length
  across all panes. Each pane clamps to its own episode length. This lets
  you compare demonstrations at the same time step even if episodes differ
  in length.
- **Async per-pane, no sync barrier.** Each pane's scrub worker decodes
  independently. Whichever finishes first displays first. No waiting for
  the slowest pane -- this keeps the UI responsive at the cost of
  momentary visual desync during fast scrubs.

### Implementation

**GridView additions (`grid.rs`):**

- `frame_slider_dragging: bool` -- tracks drag state
- `max_episode_length()` -- slider range (max total_frames across panes)
- `current_relative_frame()` -- slider position from first pane during
  playback
- `scrub_all_to(relative_frame)` -- sends scrub request to every pane's
  worker, clamping to each pane's length
- `finish_scrub(relative_frame)` -- cancels scrub, seeks all panes,
  resumes playback

**Tick update (`grid.rs`):**

Added a scrub poll path in `tick()`: when `frame_slider_dragging` is true,
polls `poll_scrub_frame()` on all panes and uploads results as textures.
Uses `ctx.request_repaint()` to keep the poll loop running.

**Slider UI (`ui/panels.rs`):**

`show_grid_frame_slider()` -- same visual style as the single-video
slider (accent-colored rail + handle). Throttled at 15 Hz using the
shared `last_scrub_seek` timer. On drag release calls `grid.finish_scrub()`
which resumes playback.

**Layout (`app.rs`):**

Added a `TopBottomPanel::bottom("grid_slider")` in the grid mode branch,
sitting between the trajectory panel and the grid footer.

### Scalability

The scrub worker architecture scales naturally because each pane is
independent -- no shared state, no locks, no sync barriers.

| Grid    | Worker threads | Decode memory | CPU     |
|---------|---------------|---------------|---------|
| 2x2 (4)  | 4             | ~40-80 MB     | No issue |
| 3x3 (9)  | 9             | ~90-180 MB    | Fine on 8+ cores |
| 4x4 (16) | 16            | ~160-320 MB   | May contend |
| 5x5+ (25+) | 25+         | ~250+ MB      | Diminishing returns |

Workers are lazy (only created on first scrub) so grid panes that never
scrub don't pay for a thread. The forward-scrub fast path (decode 2-3
frames, no seek) and drain-to-latest pattern (slow panes skip to newest
request) both help at scale.

Practical bottleneck: CPU contention with AV1 at high pane counts.
Fix would be a shared thread pool with max concurrency, but not needed
at current grid sizes.

### Result

Tested at 2x2 through 5x5 grid sizes. Scrubbing is responsive -- all
panes update frames during drag and playback resumes on release. Visual
desync between panes during fast backward scrubs is brief and
imperceptible at normal scrub speed.

## Future improvements

- **Nearest-keyframe fallback:** If decode-forward takes too long (e.g. >80ms),
  show the keyframe immediately and refine to the exact frame async.
- **Scrub frame cache:** LRU cache of recently decoded scrub frames. Useful
  when scrubbing back and forth over the same region.
- **Reduced resolution scrub:** Decode at half resolution during drag for
  speed, switch to full resolution on release.
- **Thread pool for large grids:** Cap concurrent decode threads to avoid
  CPU contention at 16+ panes.

## Files changed

- `src/video.rs` -- `decode_frame_at()` replaced with `scrub_worker()`
  persistent thread function
- `src/cache.rs` -- VideoPlayer scrub fields changed from per-request
  (`scrub_rx`/`scrub_cancel`) to persistent worker (`scrub_tx`/`scrub_result_rx`),
  lazy init in `scrub_to()`, drain-based `cancel_scrub()`
- `src/ui/panels.rs` -- throttled `scrub_to()` during single-video drag,
  new `show_grid_frame_slider()` for grid view
- `src/app.rs` -- scrub frame polling in update loop, `last_scrub_seek`
  field, grid slider panel in layout
- `src/grid.rs` -- `frame_slider_dragging`, scrub methods (`scrub_all_to`,
  `finish_scrub`, `max_episode_length`, `current_relative_frame`), scrub
  poll path in `tick()`
