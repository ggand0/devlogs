# egui vs iced: Framework Analysis for LeRobot Curation Tool

**Date:** 2026-03-05
**Context:** Evaluating whether to reuse viewskater's iced + custom event loop pipeline for the new LeRobot dataset curation tool, or start fresh with egui.

## ViewSkater Rendering Architecture (current)

The pipeline has 4 tightly coupled layers:

### 1. Custom winit event loop (`main.rs`, ~800 lines)
Bypasses iced's built-in application shell. Manually creates the wgpu Instance -> Surface -> Adapter -> Device/Queue, then drives the render loop: batch window events -> `state.update()` (iced's view/layout/diff/draw) -> `renderer.present()` -> `frame.present()`. Includes custom event batching, message queue monitoring, and slider-drag atomics bypass.

### 2. iced `program::State` (headless widget tree)
`DataViewer` implements iced's `Program` trait with Message enum / update() / view(). But the shell is entirely custom -- the `Runner` enum manages Loading -> Ready state transitions, wgpu surface reconfiguration, clipboard, and runtime/proxy for async tasks.

### 3. Custom shader widget (`widgets/shader/`)
`ImageShader` implements iced's `shader::Program` trait. Receives a `Scene` (wrapping `Arc<wgpu::Texture>`) and creates a `TexturePipeline` with raw wgpu render pipeline, bind groups, vertex/index buffers, and WGSL shaders. Textures are uploaded once and rendered as textured quads with zoom/pan transforms.

### 4. GPU image cache (`cache/gpu_img_cache.rs`)
`GpuImageCache` holds `Arc<Device>` + `Arc<Queue>`, decodes images to RGBA, optionally BC1-compresses, creates `wgpu::Texture` objects, and stores them as `CachedData::Gpu` / `CachedData::BC1` in a sliding window cache. Textures are already on the GPU when navigation happens.

## Reusability for the Curation Tool

| Component | Reusable? | Notes |
|---|---|---|
| wgpu init boilerplate | Partially | Standard wgpu setup -- any framework can do this |
| `TexturePipeline` | In principle | Generic textured-quad rendering, but deeply iced-coupled (shader::Program trait, Storage, iced Viewport) |
| GPU texture upload utils | Yes | `cache_utils.rs` functions for creating textures, uploading RGBA/BC1. Framework-agnostic wgpu code |
| Sliding window cache pattern | Conceptually | Strategy applies to video frames, but implementation is image-directory-specific |
| Custom event loop | No | Built to compensate for iced 0.13's limitations. Hardest-won code but most iced-specific |

## Why iced is a Poor Fit for the Curation Tool

The custom event loop exists because iced's Elm architecture fights continuous rendering. The curation tool needs:

- **Continuous video playback** -- decode frames at 30fps, upload as textures, display. This is continuous animation, not event-driven.
- **Scrubbing/seeking** -- needs sub-frame latency, same problem as slider drag.
- **Episode list + video + timeline** -- multi-panel layout with different update cadences.

With iced, you'd hit the same problems that led to viewskater's custom event loop and need to build another one for a different app shape.

## Why egui is the Right Fit for the Curation Tool

1. **Immediate mode = native animation loop.** egui repaints every frame by default. Displaying a new video frame is "update the texture handle, call `ui.image()`" -- no message routing, no fighting the framework.

2. **Simple texture management.** egui's `TextureHandle` -- call `ctx.load_texture()` or `tex_handle.set()` with pixel data. For video frames from a background thread:
   ```
   bg thread: decode frame -> send RGBA over channel
   main thread: recv -> tex_handle.set(rgba) -> ctx.request_repaint()
   ```
   No custom shader widget, no TexturePipeline, no Scene enum.

3. **Layout is adequate.** The curation tool needs: sidebar (episode list), central area (video), bottom bar (timeline). egui's `SidePanel` + `CentralPanel` + `TopBottomPanel` handle this directly. This is a fixed 3-panel layout, not the dynamic N-pane split that would stress egui.

4. **eframe handles the event loop.** eframe manages winit + wgpu internally. You get `fn update(&mut self, ctx: &egui::Context, _frame: &mut eframe::Frame)` -- no custom Runner enum, no manual surface configuration, no event batching. ~800 lines of main.rs you don't write.

5. **Video decoding fits channels naturally.** `ffmpeg-next` in a background thread, `std::sync::mpsc` channel to the render thread, poll in `update()`. Immediate mode means you naturally check the channel each frame without subscription plumbing.

## egui GPU Texture Caching -- Correction

Initial analysis overstated egui's texture limitations. Key clarification:

**egui's `TextureHandle` only re-uploads when you call `.set()` with new pixel data.** Holding multiple `TextureHandle`s is the equivalent of viewskater's `Arc<wgpu::Texture>` sliding window cache -- each handle keeps its texture on the GPU, and switching which one you pass to `ui.image()` is zero-cost.

```rust
// Equivalent to viewskater's sliding window of Arc<wgpu::Texture>
let cached_frames: Vec<TextureHandle> = vec![...]; // N handles, each uploaded once
ui.image(&cached_frames[current_index]); // no re-upload, just swaps the reference
```

What egui genuinely lacks vs viewskater's pipeline:
- **BC1 compression** -- egui's TextureHandle doesn't expose compressed texture formats. For 4K images this halves VRAM; for 480p video frames it's irrelevant.
- **Custom shader rendering** -- viewskater's zoom/pan/content-fit is a custom wgpu pipeline via iced's shader::Program. egui's built-in `Image` widget handles basic scaling; `egui::PaintCallback` gives raw wgpu access if needed for custom rendering.

Neither is a dealbreaker for the curation tool.

## Grid View Scenario -- 1000 Episodes

If the dataset has 1000 episodes, efficient browsing likely means a scrollable grid of video thumbnails (like a contact sheet). This is closer to viewskater's multi-image problem than the initial 3-panel MVP suggests.

**egui handles this fine:**
- `ScrollArea` + manual grid layout (allocate rects in rows) is a common egui pattern for thumbnail browsers
- Virtual scrolling: only decode/upload textures for visible cells, evict off-screen handles
- Each visible cell gets its own `TextureHandle`, updated only when a new frame decodes
- For 20-50 visible 480p thumbnails at once: ~1.2MB per frame x 50 = ~60MB VRAM, well within budget

**Where it could get harder:**
- Draggable resizable panes (viewskater-style split) require manual rect math in egui. iced's constraint-based layout handles this natively.
- But a fixed grid or fixed panel layout (sidebar + grid + timeline) is straightforward.

## Should ViewSkater Migrate to egui?

**Not now -- but the door is open.** The GPU caching and async concerns that initially seemed like blockers are overstated. egui can do both.

The real case for staying on iced is **switching cost**, not technical superiority:
- ViewSkater works. The workarounds (custom event loop, atomics bypass) are battle-tested.
- iced 0.14 may reduce the event loop boilerplate, making the current architecture less painful.
- A full rewrite is high risk for a shipping app with users.

The case for eventually considering egui:
- iced's Elm architecture required ~800 lines of custom event loop and atomics hacks to work around. That's technical debt, not a feature.
- egui's immediate mode eliminates the class of problems that led to those workarounds.
- Layout (resizable split panes) is the one thing iced does better out of the box, but it's not impossible in egui -- just manual rect math.
- egui's async story (raw channels) is simpler and more explicit than iced's Task/proxy/subscription system. Less framework magic, more direct control.

**Verdict:** Stay on iced for now, upgrade to 0.14. Use the curation tool as a real-world egui evaluation. If egui proves ergonomic there, revisit the viewskater migration question with actual experience rather than theoretical concerns.

## egui's Real Tradeoffs (honest assessment)

| Concern | Severity for curation tool | Notes |
|---|---|---|
| Layout for fixed panels (sidebar + grid + timeline) | Low | SidePanel/CentralPanel/TopBottomPanel handle this |
| Layout for draggable resizable panes | Medium | Manual rect math required, but achievable |
| GPU texture caching | Non-issue | Multiple TextureHandles = GPU-resident cache |
| Async / background loading | Low | Raw channels work fine; no framework support but the pattern is simple |
| Custom wgpu rendering (zoom/pan) | Low | PaintCallback gives raw wgpu access when needed |
| BC1 texture compression | Non-issue | Irrelevant for 480p video frames |

## Decision

- **LeRobot curation tool:** egui + eframe (new repo)
- **ViewSkater:** stay on iced, upgrade to 0.14

egui is the better starting point for the curation tool -- immediate mode is a natural fit for video playback, and GPU caching works the same way (multiple TextureHandles). The layout and async concerns are real but manageable, especially with AI-assisted implementation.

If the curation tool later needs viewskater-level layout complexity (dynamic resizable panes), that's the point to evaluate whether egui's manual approach is working or whether iced's layout engine justifies its Elm-architecture overhead.

## Implementation Plan: egui Image Viewer First

The curation tool is essentially viewskater-for-video (grid view, scrubbing, multi-pane). Rather than jumping straight to LeRobot-specific features, build a simplified image viewer in egui first. This validates all the core patterns before layering on video and dataset-specific logic.

### Phase 1: Minimal egui image viewer
- Open a directory, display images, arrow key navigation
- Basic zoom/pan (egui built-in or PaintCallback)
- Single TextureHandle, decode on navigate
- **Validates:** egui event loop feel, texture display, keyboard input

### Phase 2: Sliding window cache
- Multiple TextureHandles as GPU-resident cache
- Preload neighboring images in background thread via channels
- Navigation slider
- **Validates:** GPU caching perf, async loading pattern, whether egui's TextureHandle approach matches viewskater's Arc<wgpu::Texture> performance

### Phase 3: Grid / multi-pane layout
- Thumbnail grid with ScrollArea + virtual scrolling
- Resizable panes (manual rect math)
- **Validates:** the layout concern -- is manual rect math painful or fine?

### Phase 4: Video + LeRobot
- Swap image decoding for ffmpeg frame decoding
- Episode metadata from Parquet/JSONL
- Curation workflow (mark/delete episodes)
- **Validates:** video playback loop, full app viability

Phase 1-2 is a direct comparison point against viewskater. If it feels clean and performs well, egui is confirmed. If it's painful, we find out before investing in LeRobot-specific features.
