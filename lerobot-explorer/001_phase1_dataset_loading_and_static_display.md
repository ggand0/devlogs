# Devlog 001: MVP — Dataset Annotation Tool + Video Playback

**Date:** 2026-03-14
**Branch:** `main` (Phases 1-3), `feat/video-playback` (Phase 4)

## What was implemented

A complete LeRobot v2.1 dataset annotation tool with video playback. Load a dataset, watch episode videos auto-play, assign color-coded text prompts via keyboard shortcuts, and export annotations in LeRobot format.

## Architecture

```
src/
├── main.rs           38 lines   CLI args (clap), eframe bootstrap
├── app.rs          1207 lines   App struct, eframe::App impl, three-panel layout,
│                                keyboard/DnD handlers, episode + video navigation,
│                                playback timer, menu bar, sliders, footer
├── cache.rs         534 lines   EpisodeCache (sliding window for episode thumbnails),
│                                VideoPlayer (bounded-channel decoder thread),
│                                SliderLoader, DecodeLruCache
├── video.rs         356 lines   ffmpeg-next AV1/H264 decoder: decode_middle_frame(),
│                                decode_all_frames_sync(), frame_to_color_image()
├── dataset.rs       202 lines   LeRobot v2.1 metadata parser (info.json, episodes.jsonl,
│                                tasks.jsonl), video path builder from template
├── annotation.rs    168 lines   4 cube color prompts, episode→prompt mapping,
│                                JSON save/load, LeRobot tasks.jsonl export
├── theme.rs          59 lines   Teal dark theme (copied from viewskater-egui)
├── perf.rs           49 lines   FPS tracker (adapted from viewskater-egui)
└── build_info.rs     29 lines   Compile-time build metadata
                    2642 lines total
```

## Two-layer cache system

The app has two independent caching layers operating on different axes:

### Layer 1: EpisodeCache (episode axis — async image loading)

Adapted directly from viewskater-egui's `SlidingWindowCache`. Operates over episodes, not images in a directory. Each slot holds a single mid-episode thumbnail frame.

- Window of ±5 episodes (11 slots)
- `spawn_load()` spawns a short-lived thread per episode: opens mp4 → seeks to middle → decodes one RGBA frame → sends via unbounded `mpsc::channel` → exits
- Same thread-per-item pattern as viewskater's image decode threads
- `poll()` called every frame to collect completed decodes and upload as TextureHandles
- `navigate_forward/backward()` shifts the window and spawns load for the new edge
- `jump_to()` reinitializes the entire window (for slider release, Home/End, click)
- Episode thumbnails show instantly as placeholders while the video player loads

### Layer 2: VideoPlayer (frame axis — async video decoding)

A proper video player backed by a persistent background decoder thread and a bounded channel.

**How it works:**

```
Main thread (egui update loop)         Decoder thread
──────────────────────────           ──────────────────────
                                     Opens mp4 via ffmpeg-next
VideoPlayer::new(path, fps, 0)       Optionally seeks to start_frame
                                     Enters decode loop:
                                       for (stream, packet) in ictx.packets()
                                         decoder.send_packet(&packet)
                                         decoder.receive_frame(&mut frame)
                                         frame → swscale → RGBA ColorImage
                                         tx.send(FrameDecodeResult) ← BLOCKS if channel full
                                         check cancel flag → exit if set
                                         ctx.request_repaint()

tick_playback() every 33ms:
  player.poll_next_frame()  ───────→  try_recv() from channel
  ← Some(tex)                         (unblocks decoder to produce next frame)
  current_texture = tex
  current_frame = player.current_frame

player.seek(frame):
  cancel.store(true)         ───────→  decoder checks cancel, exits
  spawn new thread from seek position
```

**Key design: `sync_channel(30)` bounded channel**

The channel between decoder and main thread is bounded to 30 frames. This means:
- The decoder thread **blocks** when it's 30 frames ahead of the consumer
- Memory is bounded to ~36MB (30 × 480×640×4 bytes) regardless of video length
- The decoder naturally paces itself to stay ~1 second ahead of playback at 30fps
- No need for explicit flow control, ring buffers, or frame eviction
- When the consumer pulls a frame (during playback), the decoder unblocks and produces the next one

This is fundamentally different from the initial (broken) approach that decoded all 600 frames into a HashMap of TextureHandles, which consumed ~720MB and stalled after 2 seconds because frames outside the sliding window were discarded before playback reached them.

**Seeking:**

On seek (slider drag release, Home/End, arrow keys backward):
1. Set `cancel` AtomicBool flag → decoder thread checks this between frames and exits
2. Drop old channel receiver (clears buffered frames)
3. Spawn new decoder thread from the seek position
4. New thread opens the mp4, seeks via `ictx.seek(target_us, ..target_us)`, decodes forward
5. First frame to arrive via channel gets displayed

**Frame index tracking:**

Since ffmpeg doesn't provide a reliable frame counter after seeking, frame indices are calculated from PTS:
```
time_base = stream.time_base()  // e.g., 1/15360
fps = stream.rate()             // e.g., 30/1
frame_index = (pts * time_base) * fps  // rounded to nearest integer
```
For AV1 with `has_b_frames: 0`, decode order matches presentation order, so this is accurate.

## Async vs sync summary

| Operation | Sync/Async | Where | Notes |
|-----------|-----------|-------|-------|
| Episode thumbnail decode | Async | Background thread per episode | Thread-per-item, unbounded channel. Same as viewskater. |
| Video frame decode (playback) | Async | Persistent decoder thread | Bounded sync_channel(30). Blocks when 30 ahead. |
| Video frame decode (seek) | Async | New decoder thread per seek | Old thread cancelled via AtomicBool. |
| Episode thumbnail (sync fallback) | Sync | UI thread | Used only on cache miss (~6ms, imperceptible). |
| Dataset metadata parse | Sync | UI thread | <1ms for info.json + episodes.jsonl. |
| Annotation save/load | Sync | UI thread | JSON file I/O, negligible. |

## Video decoding details

### ffmpeg-next with minimal features

Used `ffmpeg-next` with only `format` + `software-scaling` features (no filter/device/resampling). Avoids needing `libavfilter-dev`. The system has FFmpeg 6.1.1 with libdav1d for AV1 software decoding.

### Mid-episode frame sampling (for EpisodeCache thumbnails)

`decode_middle_frame()` opens the mp4, seeks to `duration / 2` in AV_TIME_BASE units, decodes the first frame after the seek position. Takes **~6ms** per episode in opt-dev profile.

### Sequential frame decode (for VideoPlayer)

`decode_all_frames_sync()` opens the mp4, optionally seeks, then decodes all frames sequentially via `ictx.packets()`. Each frame is converted from yuv420p to RGBA via swscale and sent through the bounded channel. Sequential AV1 decode at 480×640 is ~2-5ms/frame — well under the 33ms display budget at 30fps.

### FrameSender trait

A small trait abstracts over `mpsc::Sender` and `mpsc::SyncSender` so `decode_all_frames_inner()` works with both bounded (VideoPlayer) and unbounded (future use) channels without code duplication.

## UI layout

Three-panel layout with egui's built-in panel system:

- `TopBottomPanel::top` — Menu bar (File: Open/Save/Export/Quit, View: Cache Overlay) + right-aligned FPS
- `SidePanel::left` — Episode list with colored annotation dots, progress counter
- `SidePanel::right` — Episode metadata (index, frames, duration, fps, camera, task) + annotation prompt cards + save button + annotation file path
- `TopBottomPanel::bottom` (×2) — Navigation slider (custom accent-colored, full-width) + monospace footer
  - Episode mode: episode slider + `ep 003 | 640x480 | 592 frames | 4 / 8`
  - Video mode: frame slider + `▶ ep 003 | 3.2s / 19.7s | frame 96 / 592`
- `CentralPanel` — Frame/video display, scaled to fit with aspect ratio preserved

## Video mode behavior

- **Default**: Video auto-plays when dataset loads or episode changes
- **Episode navigation** (←/→, click list, slider): exits current video, shows thumbnail from EpisodeCache immediately, starts new VideoPlayer for the new episode
- **Space**: play/pause
- **Escape**: exit to thumbnail-only view (for annotation focus)
- **Enter**: re-enter video mode from thumbnail view
- **Frame slider**: drag to scrub (pauses playback), release triggers seek + resume
- **Arrow keys in video mode**: forward pulls next frame from decoder, backward seeks

## Annotation workflow

1. Dataset loads → episode list populates, first episode auto-plays
2. Watch video to identify cube color being picked
3. Press 1/2/3/4 to assign prompt (red/orange/yellow/green cube)
4. Press → to advance to next episode (video auto-plays)
5. Ctrl+S to save annotations.json
6. File → Export to LeRobot to write tasks.jsonl + episodes.jsonl

## Performance

Tested against `so101_pick_place_smolvla` (8 episodes, AV1, 480×640, 30fps, ~592 frames/episode):

| Operation | Time |
|-----------|------|
| Dataset metadata parse | <1ms |
| Episode thumbnail decode (opt-dev) | ~6ms |
| Video frame sequential decode | ~2-5ms/frame |
| Video playback sustained | 30fps (decoder stays ahead) |
| Texture upload per frame | <1ms |
| Peak video memory (bounded channel) | ~36MB (30 frames) |
| Episode switch (thumbnail from cache) | <1ms |
