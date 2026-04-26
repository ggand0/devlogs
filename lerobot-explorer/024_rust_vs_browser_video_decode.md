# Devlog 024 -- Rust + egui vs browser-based video decoding in grid view

## Context

After posting tracelr on r/robotics, a commenter asked why I didn't fork
HuggingFace's official lerobot-dataset-visualizer (a Next.js/React web app).
This entry documents the reasoning and the actual technical differences for
the record.

Original motivation was intuitive: a native Rust app running locally should
be faster than a JS web app for rendering video, especially in a grid view
with many concurrent streams. No formal benchmarks were done to confirm this.

## HuggingFace lerobot-dataset-visualizer

- React / Next.js / Tailwind CSS
- Runs via Docker (port 7860) or bun dev server (port 3000)
- Has video playback, data graphs, 3D URDF viewer with EE trail rendering
- Single-episode focused, no grid view with concurrent playback

Repo: https://github.com/huggingface/lerobot-dataset-visualizer

## Where the actual difference is

### Raw decode speed

Roughly equivalent. Both call native C codecs under the hood:
- tracelr: Rust ffmpeg-next bindings -> libavcodec C code
- Browser: built-in media pipeline (ffmpeg / gstreamer / platform codecs)

Rust is not decoding the H.264 bitstream itself. The codec work is the
same native C code either way.

### Frame delivery (where the gap is)

**tracelr (Rust/egui):**
```
Thread opens file via ffmpeg (direct syscall)
  -> demuxes packets
  -> feeds packets to libavcodec decoder
  -> gets raw YUV frame
  -> swscale converts to RGBA
  -> copies RGBA buffer to egui texture (wgpu upload)
  -> GPU renders quad

x25 panes = 25 OS threads, each with its own decoder context.
```

**Browser (Next.js):**
```
React renders <video> element per grid cell
  -> browser creates media pipeline per element
  -> browser's internal thread demuxes + decodes (native codec)
  -> decoded frame goes to browser compositor
  -> compositor composites video layer with DOM layers
  -> compositor hands off to GPU for final render

x25 elements = 25 browser media pipelines.
Each frame passes through: codec -> compositor -> DOM layout -> GPU.
```

### Per-aspect comparison

| Aspect | Rust/egui | Browser |
|--------|-----------|---------|
| Decode | ffmpeg C call (direct) | Browser native codec (same speed) |
| Frame delivery | RGBA -> wgpu texture (1 copy) | Decoder -> compositor -> DOM layer -> GPU (multiple passes) |
| Seek control | Packet-level, reuse decoder context | video.currentTime, browser flushes and rebuilds internal state |
| Thread control | OS threads, full control over priority and lifetime | Browser-managed, no direct control |
| Overhead per stream | One ffmpeg context + one texture | Media pipeline + compositor layer + DOM node + GC pointers |
| Scaling to 25 streams | Linear, just more threads | Compositor overhead grows, layering 25 video elements is expensive |

### Scrubbing

tracelr's decode_frame_at() uses a persistent decoder context per video
player. Forward scrubs resume reading packets without seeking (2-3 frames
of decode). Backward scrubs seek to the nearest keyframe and decode forward.
The decoder context stays open across scrubs.

Browser video.currentTime triggers a full internal seek each time. The
browser flushes decoder state, seeks in the container, reinitializes, and
decodes forward. Higher latency per seek, no reuse of decoder state.

## Why Rust was chosen

Gut feeling, not benchmarks. For a desktop app that renders video frames
in a grid, the native path (ffmpeg decode -> raw buffer -> GPU texture)
has fewer intermediaries than the browser path. At 25 concurrent streams
the overhead difference compounds. The grid view is the core feature of
tracelr, so optimizing for that case drove the tech choice.

If someone asks for hard numbers, the honest answer is that no A/B
benchmarks were done. The intuition is that native apps are faster for
rendering-heavy workloads, and the architectural comparison above supports
that, but the actual measured difference at e.g. 3x3 or 5x5 grids is
unknown.

## Other reasons for Rust besides performance

- Single binary, no Docker / Node / bun setup
- Direct filesystem access (load dataset dirs, no upload step)
- Already familiar with Rust + egui from viewskater-egui
