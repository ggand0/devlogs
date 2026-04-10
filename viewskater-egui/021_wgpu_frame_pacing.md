# 021 wgpu frame pacing investigation and fix

Branch: `feat/renderer-selection`

## Summary

Investigated and fixed wgpu's micro-stutter during keyboard navigation of 4K image sequences. The root cause was variable GPU pipeline timing in wgpu's multi-stage submission model. Fixed by adding explicit GPU synchronization via `device.poll(Maintain::Wait)` at the start of each frame.

## Problem

After enabling both glow and wgpu backends (devlog 020), keyboard navigation ("skate mode") through cached 4K images felt noticeably less smooth on wgpu compared to glow, despite identical FPS numbers (61 UI / 11.5 image FPS). The issue manifested as irregular frame spacing — not dropped frames, but inconsistent temporal delivery that creates a perception of micro-stutter.

Slider navigation felt identical in both backends, because the ~90ms CPU decode time per frame dominates over any rendering pipeline variance.

## Why egui defaults to glow (and the shift to wgpu)

### Historical default: glow

eframe 0.31.1 includes `glow` in its default Cargo features but not `wgpu`. When both features are enabled, the `Renderer::default()` impl actually prefers wgpu (the comment says "if the user added wgpu they probably wanted to use it"). But since most users just write `eframe = "0.31"`, they get glow.

Reasons glow was the original default:
- Lightweight wrapper around OpenGL 2.0+ with few dependencies and fast compile times
- Broader hardware compatibility historically
- Lower overhead — simpler initialization pipeline

### eframe 0.34.0: wgpu becomes default

As of eframe 0.34.0 (2026-03-26), wgpu replaced glow as the default renderer (egui PR #7615 by @emilk). The rationale from GitHub issue #5889:

1. **Community convergence** — the Rust graphics ecosystem is standardizing on wgpu
2. **Maintenance burden** — two backends means duplicated code and scattered developer effort
3. **Forcing function** — making wgpu default incentivizes fixing wgpu-specific issues
4. **Modern API coverage** — Vulkan, Metal, DX12, and WebGPU from a single abstraction

This means our investment in fixing wgpu issues is aligned with egui's direction.

## Known wgpu latency issues in egui

Multiple GitHub issues document frame pacing problems with wgpu, several Linux-specific:

| Issue | Description | Platform |
|-------|-------------|----------|
| [emilk/egui#5958](https://github.com/emilk/egui/issues/5958) | ~100ms lag spike on any mouse button press | General |
| [emilk/egui#5037](https://github.com/emilk/egui/issues/5037) | High input latency, worse on Vulkan than DX12 | Cross-platform |
| [gfx-rs/wgpu#6548](https://github.com/gfx-rs/wgpu/issues/6548) | Latency on (Arch) Linux but not Windows | Linux |
| [emilk/egui#3924](https://github.com/emilk/egui/issues/3924) | eframe unusable with wgpu on Wayland | Linux/Wayland |
| [emilk/egui#7718](https://github.com/emilk/egui/issues/7718) | wgpu Vulkan backend hangs on resize | Windows |

The Vulkan backend on Linux appears particularly affected, which matches our environment.

## Root cause analysis

### egui-wgpu rendering pipeline

The `paint_and_update_textures()` function in `egui-wgpu/src/winit.rs` runs this sequence each frame:

```
1. update_texture()    — queue.write_texture() for each changed texture
2. update_buffers()    — queue.write_buffer_with() for vertex/index data
3. get_current_texture — acquire swap chain image (may block for vsync)
4. render()            — record draw commands into command encoder
5. queue.submit()      — submit EVERYTHING in a single batch
6. present()           — present the frame
```

The critical issue: **all texture uploads and render commands are batched into a single `queue.submit()` call**. Texture uploads do not get submitted early — they wait for the full render pipeline to be encoded.

### queue.write_texture() is deferred, not immediate

`queue.write_texture()` does not execute immediately. It:
1. Allocates an internal staging buffer
2. Copies CPU pixel data into the staging buffer
3. Records an internal copy command from staging buffer to GPU texture
4. The actual GPU copy only executes at the next `queue.submit()`

For a 4K RGBA image, each write_texture call stages 31.6 MB (3840 x 2160 x 4 bytes). The staging buffer allocation and internal scheduling can vary frame-to-frame.

### Comparison with glow

**glow (OpenGL):**
```
1. glTexImage2D()   — synchronous, driver-optimized, data is in GPU hands immediately
2. Draw calls       — immediate mode
3. glSwapBuffers()  — blocks until vsync, provides consistent frame pacing
```

`glSwapBuffers` acts as a natural synchronization point — the CPU blocks until the GPU finishes and vsync fires. This gives perfectly even ~16.6ms frame spacing.

**wgpu (Vulkan):**
The multi-stage pipeline (staging allocation → copy command recording → batch submission → GPU execution → presentation) has more variable per-frame timing. Without an explicit sync point, frames can finish at irregular intervals even when the average throughput is the same.

### Why slider navigation is unaffected

Slider scrubbing jumps to arbitrary frames, triggering a full image decode (~90ms for 4K PNG). This decode time completely dominates the rendering pipeline variance (~0.1-1ms). The micro-stutter is only perceptible during rapid keyboard navigation where cached images are swapped every frame with no decode overhead.

## Fix: explicit GPU synchronization

### Implementation

Added `device.poll(Maintain::Wait)` at the start of each `update()` call in `src/app.rs`:

```rust
fn update(&mut self, ctx: &egui::Context, frame: &mut eframe::Frame) {
    // Synchronize with the GPU before building the next frame.
    // Without this, wgpu's multi-stage pipeline (staging buffer → copy →
    // submit → present) can finish at variable times, causing irregular
    // frame spacing during rapid keyboard navigation of 4K images.
    // Blocking here until the previous frame's GPU work completes gives
    // deterministic frame pacing comparable to glow's glSwapBuffers.
    // Returns None when using the glow backend, so this is a no-op there.
    if let Some(render_state) = frame.wgpu_render_state() {
        render_state.device.poll(eframe::wgpu::Maintain::Wait);
    }
    // ...
}
```

### How it works

`device.poll(Maintain::Wait)` blocks the CPU thread until all previously submitted GPU work (from the prior frame) has completed. This forces CPU and GPU into lockstep:

```
Frame N:   update() → [eframe renders → submit → present]
                ↓
Frame N+1: device.poll(Wait) ← blocks until frame N's GPU work finishes
           update() → [eframe renders → submit → present]
```

This replicates the synchronization model that glow gets implicitly from `glSwapBuffers`. The tradeoff is we give up CPU/GPU overlap (the CPU can't start building frame N+1 while the GPU is still rendering frame N), but for an image viewer running at 60 FPS this is negligible.

### Why it's safe

- `frame.wgpu_render_state()` returns `None` when using the glow backend, so the poll is automatically a no-op for OpenGL
- `device.poll(Wait)` is a standard wgpu API — it processes completed work and invokes any pending callbacks
- No risk of deadlock: we're polling for completion of already-submitted work, not waiting for something that hasn't been submitted yet

## Other approaches tried

### PresentMode::Mailbox — FAILED

```rust
let wgpu_options = egui_wgpu::WgpuConfiguration {
    present_mode: wgpu::PresentMode::Mailbox,
    ..Default::default()
};
```

Caused wgpu to fail silently on Linux (Vulkan) — the application launched and immediately exited with no error output. Mailbox mode is not universally supported. Reverted to the default `AutoVsync`.

### desired_maximum_frame_latency: Some(1) — kept but insufficient alone

```rust
let wgpu_options = egui_wgpu::WgpuConfiguration {
    desired_maximum_frame_latency: Some(1),
    ..Default::default()
};
```

Reduces the presentation queue depth from default 2 to 1 frame. This is kept in the configuration as it reduces overall latency, but alone it did not resolve the frame pacing irregularity.

## Other potential approaches (not yet tried)

### Separate texture upload submission

The most architecturally correct fix would be to split egui-wgpu's `paint_and_update_textures()` into two phases:
1. Submit texture uploads in their own `queue.submit()` call
2. Then encode and submit render commands separately

This would let the GPU start copying texture data while the CPU encodes render commands. However, this requires modifying egui-wgpu internals (forking or upstreaming a change).

### Triple-buffered texture slots

From [wgpu discussion #5899](https://github.com/gfx-rs/wgpu/discussions/5899): rotating between 3 texture slots so the GPU is never reading from and writing to the same texture in the same frame. This decouples the write/read pipeline but adds memory overhead (3x GPU texture memory per cached image).

### Compute shader texture copy

Copy texels from a buffer to a texture via compute shader instead of write_texture. Keeps the work entirely on the GPU side, avoiding CPU-side staging buffer allocation variance. High complexity for uncertain benefit.

## Linux test results after fix

Tested on Linux with `device.poll(Maintain::Wait)` + `dithering: false` + `desired_maximum_frame_latency: Some(1)`.

### 4K PNG sequences (3840x2160, ~10 MB each, 140 frames)

| | glow | wgpu |
|---|---|---|
| UI FPS | 61 | 61-62 |
| Image FPS | 11.5 | 11.5 |
| Keyboard nav feel | Smooth | Smooth (identical after fix) |
| Slider nav feel | Smooth | Smooth |
| Visual quality | Pixel-crisp | Pixel-crisp (after dithering: false) |

### 1080p PNG sequences and small images

Both backends identical — no perceptible difference in FPS, visual quality, or navigation feel.

## macOS test results (MacBook Pro M1, late 2020)

Tested with 4K PNG sequences.

- wgpu (Metal) renders **slightly smoother** than glow (OpenGL) for keyboard navigation
- This is expected: macOS deprecated OpenGL (frozen at 4.1, no longer actively optimized by Apple), while Metal is Apple's native GPU API, heavily optimized for Apple Silicon
- M1's unified memory architecture likely eliminates the staging buffer overhead that caused frame pacing issues on Linux/Vulkan — Metal can access CPU memory more directly

**Result: wgpu is the better backend on macOS.**

## Windows test results

Pending. wgpu maps to DX12 on Windows (Vulkan as fallback). Expected to match or beat glow:
- GPU vendors (NVIDIA, AMD, Intel) actively maintain both OpenGL and DX12 drivers on Windows
- egui issue #5037 noted wgpu latency was lower on DX12 than on Vulkan
- The `device.poll(Wait)` fix provides insurance against frame pacing variance

## Cross-platform summary

| Platform | wgpu backend | wgpu vs glow |
|----------|-------------|--------------|
| Linux | Vulkan | Matches glow (after device.poll fix) |
| macOS | Metal | Smoother than glow (OpenGL deprecated) |
| Windows | DX12 | Pending (expected to match or beat) |

### Conclusion

wgpu matches or beats glow on all tested platforms. The `device.poll(Wait)` fix resolved the only case where wgpu was worse (Linux keyboard nav with 4K images).

**Decision:** Switch to wgpu-only after verifying on Windows. Rationale:
- Frame pacing parity achieved with glow
- eframe 0.34.0 already made wgpu the default upstream
- Eliminates 6 MB binary size overhead from carrying both backends
- One backend to maintain instead of two
- More familiar from iced version of the app

## Key files

- `src/main.rs` — renderer CLI arg, `NativeOptions` with `dithering: false` and `wgpu_options`
- `src/app.rs` — `device.poll(Maintain::Wait)` in `update()` for frame pacing
- `Cargo.toml` — eframe features with both `glow` and `wgpu` enabled
- `devlogs/020_renderer_selection.md` — initial renderer selection implementation and findings
