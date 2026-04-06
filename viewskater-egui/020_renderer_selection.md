# 020 Renderer selection: glow (OpenGL) vs wgpu

Branch: `feat/renderer-selection`

## Summary

Added support for both glow (OpenGL) and wgpu rendering backends via a `--renderer` CLI flag. Default remains glow. Investigated visual and performance differences between the two backends on Linux with 4K PNG sequences.

## Motivation

The app uses eframe 0.31 which defaults to glow (OpenGL). OpenGL textures are managed at the driver level — after closing a directory, ~645 MB of residual GPU memory is retained by the driver's internal texture pool (not a leak; reused on next open). There is no GL API to force release this memory.

wgpu (Vulkan/Metal/DX12) gives more explicit control over GPU resource lifetimes through its API. Supporting both backends allows comparing memory behavior, visual quality, and performance to decide which is better suited for an image viewer.

## Implementation

### Cargo.toml

Changed eframe dependency from default features to explicit feature selection with both backends enabled:

```toml
eframe = { version = "0.31", default-features = false, features = [
    "accesskit",
    "default_fonts",
    "glow",
    "wgpu",
    "wayland",
    "x11",
] }
```

### CLI flag

Added `--renderer` argument via clap in `src/main.rs`:

```rust
#[arg(long, default_value = "glow")]
renderer: String,
```

Maps to `eframe::Renderer::Glow` or `eframe::Renderer::Wgpu` and passed through `NativeOptions`.

Usage:
```
cargo run --profile opt-dev -- --renderer wgpu
cargo run --profile opt-dev -- --renderer glow   # default
```

### wgpu configuration

```rust
let wgpu_options = egui_wgpu::WgpuConfiguration {
    desired_maximum_frame_latency: Some(1),
    ..Default::default()
};

let options = eframe::NativeOptions {
    viewport,
    renderer,
    dithering: false,
    wgpu_options,
    ..Default::default()
};
```

- **`dithering: false`** — eframe's wgpu backend applies interleaved gradient noise dithering by default (shader-level, in `egui.wgsl`). This makes images look slightly soft compared to glow's pixel-crisp output. Controlled via a `dithering` uniform in the fragment shader; setting `false` passes `0` so the shader skips the noise. No effect on glow (it doesn't have shader-level dithering).
- **`desired_maximum_frame_latency: Some(1)`** — reduces the wgpu presentation queue depth from default 2 to 1 frame, reducing input-to-display latency.
- **`PresentMode::Mailbox`** was tested but caused wgpu to fail silently on Linux (Vulkan). Reverted to the default `AutoVsync`.

### Binary size impact

| Build | Backend(s) | Size (opt-dev) |
|-------|-----------|----------------|
| main (glow only) | glow | 22 MB |
| this branch | glow + wgpu | 28 MB |

+6 MB (~27%) increase. Acceptable for having both backends available. Would shrink with LTO in release builds.

## Testing: glow vs wgpu comparison

Test data: `4k_PNG_10MB` — 140 frames, 3840x2160 PNG, ~10 MB each.
System: Linux, opt-dev profile.

### FPS

| Backend | UI FPS | Image FPS |
|---------|--------|-----------|
| glow    | 61     | 11.5      |
| wgpu    | 61-62  | 11.5      |

Numerically identical performance.

### Visual quality

After disabling dithering (`dithering: false`), individual frames look identical between backends. Before the fix, wgpu images appeared slightly soft due to the interleaved gradient noise applied in the fragment shader.

### Keyboard navigation feel

**glow**: Smooth, consistent frame pacing during arrow key navigation. No perceptible jitter — feels like a 3D game rendering every frame at a locked rate.

**wgpu**: Subtle but noticeable inconsistency in frame delivery timing during keyboard navigation. Not dropped frames (FPS numbers match), but irregular frame pacing — as if frames are not evenly spaced in time. This creates a perception of micro-stutter or jitter even at the same measured FPS.

Setting `desired_maximum_frame_latency: Some(1)` did not resolve this. The issue may be related to:

1. **wgpu's command buffer submission model** — wgpu records commands into a command buffer, submits to the GPU queue, then presents. This multi-stage pipeline may introduce variable latency per frame compared to glow's more direct `glSwapBuffers` path.
2. **Texture upload path** — wgpu uses `queue.write_texture()` which internally manages staging buffers. glow does direct `glTexImage2D`. The staging buffer path may have more variable timing.
3. **Vulkan driver frame pacing** — Vulkan's swap chain presentation may have different timing characteristics than GLX/EGL on the same hardware.
4. **eframe's vsync measurement** — eframe measures vsync time at both `surface.get_current_texture()` and `output_frame.present()`. Variable wait times at these points would cause uneven frame delivery.

### Slider navigation

Slider scrubbing feels identical in both backends. This makes sense — slider navigation is dominated by CPU decode time (~90ms per 4K PNG), so the rendering backend's frame pacing differences are masked by the much larger decode latency.

## Architecture notes

### Texture pipeline: glow vs wgpu

**glow path:**
1. `egui::ColorImage` → `ctx.load_texture()` or `TextureHandle::set()`
2. egui-glow calls `glTexImage2D` — synchronous GPU upload
3. `glSwapBuffers` — direct buffer swap

**wgpu path:**
1. `egui::ColorImage` → `ctx.load_texture()` or `TextureHandle::set()`
2. egui-wgpu calls `queue.write_texture()` — writes to internal staging buffer, copies to GPU texture
3. Command buffer recorded → submitted to GPU queue → `output_frame.present()`

The wgpu path has more pipeline stages, which may explain the frame pacing variance.

### Root cause of wgpu keyboard navigation micro-stutter

The most likely cause is the texture upload path difference. In skate mode (keyboard navigation through cached images), `TextureHandle::set()` uploads ~31.6 MB of pixel data (3840×2160×4 bytes) to the GPU on nearly every frame.

**glow**: `glTexImage2D` is a single call that hands pixel data directly to the driver. OpenGL drivers have decades of optimization for this exact operation — the upload completes before the next draw call with deterministic timing.

**wgpu**: `queue.write_texture()` allocates an internal staging buffer, copies the 31.6 MB into it, records a buffer-to-texture copy command, then submits the whole command buffer to the GPU queue. The staging buffer allocation and reuse can vary frame-to-frame, and the GPU must execute the copy *then* draw within a single submitted batch.

Every frame has a 31.6 MB upload where glow takes one predictable path and wgpu takes a multi-step path with variable internal allocation. That variance shows up as irregular frame spacing — micro-stutter even though the average FPS is identical.

Slider navigation feels identical in both backends because the ~90ms CPU decode time per frame completely drowns out any upload timing difference.

### Dithering details

The wgpu shader (`egui.wgsl`) uses interleaved gradient noise (Jimenez 2014, "Next Generation Post-Processing in Call of Duty"). It quantizes to 256 levels and adds noise scaled by 0.95 to prevent dithering of flat colors. The noise is position-dependent (uses fragment position) to create a stable dithering pattern. Controlled by a `dithering: u32` uniform — `1` enables, `0` disables.

For an image viewer displaying photographic content, this dithering is counterproductive — it adds noise to already-dithered source images and softens pixel-level detail.

## glow vs wgpu: pros and cons for an image viewer

### glow (OpenGL)

**Pros:**
- Smoother keyboard navigation — deterministic frame pacing from direct `glTexImage2D` uploads
- Pixel-crisp rendering out of the box (no shader-level dithering)
- Smaller binary size (22 MB vs 28 MB with both enabled)
- Mature driver support — OpenGL drivers are battle-tested across decades of hardware
- Simpler pipeline — fewer stages between CPU and displayed pixels means fewer sources of timing variance

**Cons:**
- No explicit GPU memory control — textures are managed by the driver's internal pool. After closing a directory, ~645 MB of residual GPU memory is retained and cannot be force-released via any GL API
- OpenGL is a legacy API — no longer receiving significant driver investment from GPU vendors. Vulkan/Metal/DX12 are the focus
- No compute shader support in egui-glow for potential future features (e.g. GPU-side image processing)
- Platform support narrowing — macOS deprecated OpenGL (still works but frozen at 4.1), some embedded/web targets prefer wgpu

### wgpu (Vulkan/Metal/DX12)

**Pros:**
- Explicit GPU resource management — textures can be precisely allocated and freed, avoiding the driver-level memory pooling issue
- Modern API with active driver development from all GPU vendors
- Cross-platform abstraction — single API maps to Vulkan (Linux/Windows/Android), Metal (macOS/iOS), DX12 (Windows), and WebGPU (browsers)
- Compute shader access via egui-wgpu for potential GPU-side image processing
- Better long-term viability as OpenGL is deprecated on some platforms

**Cons:**
- Micro-stutter during rapid 4K texture uploads — the staging buffer → copy → submit pipeline has variable per-frame timing compared to glow's direct upload
- Requires `dithering: false` to match glow's pixel-crisp output (default dithering softens images)
- +6 MB binary size increase
- More complex pipeline — more stages means more potential sources of frame timing variance
- `PresentMode::Mailbox` not supported on all Linux Vulkan drivers (caused silent launch failure)

### Verdict

For an image viewer that prioritizes smooth navigation through high-resolution image sequences, glow is the better default. The direct texture upload path gives inherently more consistent frame pacing during the app's primary use case (rapid keyboard navigation through cached 4K images).

wgpu's advantages (explicit memory control, modern API, compute shaders) are real but less relevant today — the app doesn't need GPU compute, and the driver memory pooling is a cosmetic issue (memory is reused, not leaked). If macOS drops OpenGL support entirely or GPU-side processing becomes needed, wgpu is available as a fallback via `--renderer wgpu`.

## Current status

- Both backends functional
- glow remains the better default for an image viewer (pixel-crisp, smoother keyboard nav)
- wgpu available via `--renderer wgpu` for comparison and future evaluation
- Open question: whether wgpu's frame pacing can be improved, or if the architectural difference makes glow inherently better for this use case

## Key files

- `src/main.rs` — renderer CLI arg, `NativeOptions` with `dithering` and `wgpu_options`
- `Cargo.toml` — eframe features with both `glow` and `wgpu` enabled
