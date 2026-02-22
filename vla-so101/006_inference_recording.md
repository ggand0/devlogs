# Inference Script — Recording Implementation

## Overview

`scripts/infer.py` runs a trained SmolVLA policy on the SO-101 follower arm with optional video recording for demos.

## Recording Architecture

Recording is fully decoupled from the control loop via a background `FrameRecorder` thread.

### Why background thread?

Earlier attempts wrote frames inline from the main thread. This caused two problems:
- **Frame jumps**: Any pause in the main thread (log_say TTS, phase transitions, inference variability) meant no frames were captured, creating visible discontinuities.
- **Audio desync**: Audio (captured in real-time by ffmpeg) drifted ahead of video because missed frames shortened the video duration. In one recording, video was 82s but audio was 103s — 21s of drift from ~630 missing frames.

### How it works

1. A single `ffmpeg` subprocess receives raw RGB video frames on stdin AND captures audio from the webcam mic via PipeWire simultaneously. ffmpeg timestamps both streams against its own clock, guaranteeing sync.

2. `FrameRecorder` runs a daemon thread that reads camera frames via `async_read()`, stitches them (vertical concat), and pipes to ffmpeg. This runs continuously regardless of what the control loop is doing — during resets, waits, inference, TTS calls, everything.

3. Wall-clock frame duplication keeps video locked to real time: the thread tracks `time.monotonic()` and duplicates the latest frame to fill any gap when capture falls behind. This prevents A/V desync even when camera reads or GIL contention cause jitter.

4. PiP compositing (scaling, cropping, overlay) is done entirely by ffmpeg's `filter_complex` in native C — the Python thread only does `async_read()` + `np.concatenate`.

5. The control code (`reset_to_home`, `run_episode`, etc.) has zero knowledge of recording. No recording parameters, no conditional frame writes.

6. Shutdown: `recorder.stop()` → `ffmpeg_proc.stdin.close()` → `ffmpeg_proc.terminate()` (SIGTERM) → `ffmpeg_proc.wait()`. SIGTERM is needed because the live PipeWire audio input never sends EOF, so ffmpeg won't exit on its own. ffmpeg handles SIGTERM gracefully — writes the mp4 trailer and exits cleanly.

### Video layout

960×540 (16:9) PiP:
- Overhead cam (640×480) scaled to 960×720, center-cropped to 540 height (fills background)
- Wrist cam (640×480) center-cropped to 480×480 square, scaled to 240×240, overlaid bottom-right with 2px white border

### Audio

Webcam mic (Logitech C920) captured via PipeWire (`alsa_input.usb-046d_HD_Pro_Webcam_C920-02.analog-stereo`). Direct ALSA access (`hw:2,0`) doesn't work because OpenCV holds the device — PipeWire multiplexes access.

Note: C920 mic quality is poor (tinny, picks up servo noise). For X posts, the no-audio version is better.

### Codec

H.264 (`libx264 -preset ultrafast`) + AAC 128kbps. The `ultrafast` preset keeps CPU overhead low since encoding happens in real-time alongside inference.

## CLI Flags

- `--record` — Enable recording. Optional path arg, defaults to `recordings/infer_{timestamp}.mp4`
- `--direct-reset` — Skip snap-to-zeros step, interpolate straight to home position (smoother on video)
- `--episode-time` — Episode duration in seconds (default 10)
- `--num-rollouts` — Number of episodes

## TTS Announcements

Uses lerobot's `log_say`:
- "Resetting arm" — before reset
- "Starting episode N" — before each episode
- "Episode ended" — after each episode

## Bugs Fixed

### `_interval` → `_fps` attribute error (root cause of 0-byte videos)

`FrameRecorder.__init__` stored `self._interval = 1.0 / fps` but `_capture_loop` referenced `self._fps`. The thread crashed immediately on first iteration with `AttributeError`, silently caught by a bare `except`. Zero frames were ever written. This was the root cause of all 0-byte videos and apparent "desync" in every test after the background thread was introduced — the thread was never actually running.

Fixed by changing to `self._fps = fps` and logging exceptions instead of swallowing them.

### A/V desync from inline frame writing

Before the background thread, frames were written inline from the main control loop. Any pause (TTS, phase transitions, inference variability) meant no frames captured, shortening video duration relative to audio. One recording had 82s video vs 103s audio — 21s drift from ~630 missing frames.

Fixed by the background `FrameRecorder` thread with wall-clock frame duplication.

### ffmpeg hang on shutdown

`ffmpeg_proc.wait()` hung forever because the live PipeWire audio input never sends EOF. Fixed with `terminate()` (SIGTERM) — ffmpeg handles it gracefully, writes the mp4 trailer.

### ALSA device busy

Direct ALSA access (`hw:2,0`) failed because OpenCV already held the webcam device. Fixed by using PipeWire source name, which multiplexes access.

## Verified Working

- `recordings/infer_20260217_231615.mp4` — multi-episode recording with no desync or severe lags. Confirmed wall-clock frame duplication and background thread approach works correctly.
