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

Tested with the same 4K PNG sequences. wgpu maps to DX12 on Windows.

### Performance and smoothness

| | glow | wgpu |
|---|---|---|
| Rendering smoothness | Smooth | Smooth (identical) |
| Keyboard nav FPS | baseline | +1-2 FPS |
| Slider nav | Smooth | Smooth |
| Visual quality | Pixel-crisp | Pixel-crisp |

wgpu matches glow and may be marginally faster (1-2 FPS) for keyboard navigation. No frame pacing issues observed.

### Memory usage difference

This is the one significant concern from Windows testing.

| State | glow RSS | wgpu RSS | Delta |
|-------|----------|----------|-------|
| After loading initial 4K images (cache 5-1-5) | 81 MB | 448 MB | **+367 MB** |
| After scrubbing slider under 1024 MB LRU budget | 1.1 GB | 1.4 GB | **+300 MB** |

Both backends report the same logical cache content in the FPS overlay (`L:1012, C:190`), so the difference is in untracked GPU-side allocations.

### Why wgpu uses more memory

**Initial 367 MB gap (startup overhead):**
- wgpu links significantly more code than glow (backend abstraction layer, shader translator (naga), resource tracker, validation layer)
- DX12 allocates more upfront state than OpenGL: descriptor heaps, command allocators, pipeline state objects, root signatures
- OpenGL drivers defer most of this allocation lazily
- This is fixed overhead — does not scale with image count

**Steady-state 300 MB gap (post-scrubbing):**
1. **Staging buffer retention** — `queue.write_texture()` allocates an internal staging buffer for each upload. wgpu retains these in a pool for reuse. Each new image scrubbed past may allocate a fresh staging buffer. glow's `glTexImage2D` does not need staging buffers on desktop GPUs.
2. **DX12 resource tracker bookkeeping** — wgpu maintains per-resource state tracking for validation and synchronization that scales with active texture count.
3. **Texture descriptor duplication** — each `TextureHandle::set()` may leave orphaned descriptors until wgpu's internal cleanup runs.

On a 1024 MB budget, +300 MB is ~30% overhead — real but not catastrophic for typical desktops. For very memory-constrained users it matters more.

## Cross-platform summary

| Platform | wgpu backend | wgpu vs glow (perf) | wgpu vs glow (memory) |
|----------|-------------|---------------------|------------------------|
| Linux | Vulkan | Matches (after device.poll fix) | Not measured |
| macOS | Metal | **Smoother** than glow | Not measured |
| Windows | DX12 | Matches, slightly faster (+1-2 FPS keyboard nav) | **+367 MB baseline, +300 MB steady-state** |

### Conclusion

wgpu matches or beats glow on rendering performance across all tested platforms. The `device.poll(Wait)` fix resolved Linux keyboard nav frame pacing. macOS sees wgpu (Metal) as the better backend due to OpenGL deprecation. Windows performance is identical to slightly better, with the tradeoff being meaningful memory overhead.

**Decision:** Switch to wgpu-only is justified by performance, future-proofing, and macOS results. The memory overhead is the one remaining concern — investigation pending before final switch.

Rationale for switching:
- Frame pacing parity (Linux) or improvement (macOS) achieved
- Identical performance with potential slight gains on Windows
- eframe 0.34.0 already made wgpu the default upstream
- Eliminates 6 MB binary size overhead from carrying both backends
- One backend to maintain instead of two
- More familiar from iced version of the app

Open question: can the +300 MB steady-state overhead on Windows be reduced through wgpu configuration (MemoryHints, explicit polling, cache tuning)?

## Memory mitigation: MemoryHints tuning

### Background

wgpu uses [`gpu_allocator`](https://crates.io/crates/gpu-allocator) under the hood for both Vulkan (Linux) and DX12 (Windows) backends. The allocator's block size is controlled by `wgpu::MemoryHints` passed at device creation. eframe defaults to `MemoryHints::default()` which is `Performance` mode.

The wgpu-hal source at `wgpu-hal/src/dx12/suballocation.rs` (and the Vulkan equivalent) maps the hints to gpu_allocator block sizes:

```rust
match memory_hints {
    MemoryHints::Performance => gpu_allocator::AllocationSizes::default(),  // ~256+ MB blocks
    MemoryHints::MemoryUsage => gpu_allocator::AllocationSizes::new(8 * mb, 4 * mb),
    MemoryHints::Manual { suballocated_device_memory_block_size } => {
        let device_size = suballocated_device_memory_block_size.start;
        let host_size = device_size / 2;
        gpu_allocator::AllocationSizes::new(device_size, host_size)
    }
}
```

Note: `Manual` only uses `.start` of the range, ignoring `.end`.

### Bypass: WgpuSetup::Existing

eframe 0.31's `WgpuConfiguration` does not expose `device_descriptor` for direct customization. The escape hatch is `WgpuSetup::Existing`: we create the wgpu Instance, Adapter, Device, and Queue ourselves (with custom MemoryHints), then hand them to eframe via `WgpuConfiguration.wgpu_setup`.

Implementation in `src/main.rs`:

```rust
fn build_low_memory_wgpu_setup() -> egui_wgpu::WgpuSetupExisting {
    const MB: u64 = 1024 * 1024;
    let instance = wgpu::Instance::new(&wgpu::InstanceDescriptor { /* ...eframe defaults... */ });
    let adapter = pollster::block_on(instance.request_adapter(&wgpu::RequestAdapterOptions {
        power_preference: wgpu::PowerPreference::HighPerformance,
        ..
    })).expect("...");
    let (device, queue) = pollster::block_on(adapter.request_device(
        &wgpu::DeviceDescriptor {
            required_limits: wgpu::Limits { max_texture_dimension_2d: 8192, ..base_limits },
            memory_hints: wgpu::MemoryHints::Manual {
                suballocated_device_memory_block_size: (64 * MB)..(128 * MB),
            },
            ..
        },
        None,
    )).expect("...");
    egui_wgpu::WgpuSetupExisting { instance, adapter, device, queue }
}
```

### Linux test results (4K PNG sequences)

Tested with the same 4K dataset on Linux. Numbers measured at app launch and after loading the 4K directory (no navigation, sliding window cache shows `C:348` — 11 textures × 31.6 MB).

| Config | Launch RSS | After 4K load | Skate-mode keyboard nav |
|--------|-----------|---------------|--------------------------|
| glow (default) | 81 MB | (not measured) | 61 FPS UI (baseline) |
| wgpu Performance (256+ MB blocks) | 315 MB | 999 MB | 61-62 FPS UI |
| wgpu MemoryUsage (8/4 MB blocks) | 195 MB | 752 MB | **30 FPS UI (broken)** |
| wgpu Manual { 64..128 MB } | 244 MB | 872 MB | 64-65 FPS UI (no regression) |

### Why MemoryUsage broke navigation

A 4K RGBA texture is 31.6 MB (3840 × 2160 × 4). With 8 MB blocks, the texture is larger than the allocation block, forcing gpu_allocator to make a dedicated allocation per texture. Combined with the per-frame `device.poll(Wait)` synchronization, background texture uploads in the sliding window cache become slow enough to halve the UI framerate during keyboard navigation.

### Why Manual { 64..128 MB } works

64 MB blocks fit two 4K textures via sub-allocation. The allocator can reuse blocks across multiple texture allocations, avoiding the dedicated-allocation overhead of MemoryUsage mode. ~4x smaller than the default ~256 MB blocks, so memory waste is reduced without crossing the texture-size threshold.

**Savings vs Performance mode (no perf cost):**
- Launch: -71 MB (-23%)
- After 4K load: -127 MB (-13%)

**Remaining gap to glow:**
- Launch: 244 MB vs 81 MB (+163 MB)
- This is fixed wgpu/Vulkan runtime overhead that cannot be tuned away — the cost of carrying a modern explicit graphics API.

### What the memory display actually measures (and what L/C mean)

While analyzing the MemoryHints results above, it became clear that the FPS overlay's memory display has a UI clarity issue worth documenting.

The overlay shows: `Img: 12.3 FPS | 234 MB (L:180 C:45)`

**The 234 MB number** is process RSS (Resident Set Size) from the `sysinfo` crate, polled once per second. RSS is what the OS reports as the process's resident physical memory. What it includes depends on the platform:

| Platform | What's in RSS |
|----------|---------------|
| Linux (discrete GPU) | CPU heap + host-visible GPU buffers (staging, uniforms, mapped textures). **Pure VRAM textures: NOT counted.** |
| Windows (discrete GPU) | Same as Linux |
| macOS Apple Silicon | **Everything** — unified memory means GPU and CPU share the same RAM, all in RSS |
| Windows/Linux integrated GPU | Same as Apple Silicon — shared RAM, all in RSS |

**L (LRU decode cache):** Real CPU heap memory. `HashMap<usize, egui::ColorImage>` storing decoded RGBA pixel data. Always counted in RSS. The `L:` number accurately reflects what's in process memory.

**C (sliding window Cache):** Logical texture size computed as `width × height × 4` summed across loaded textures. **NOT counted in RSS on discrete GPUs** — the actual texture pixel data lives in VRAM, only the small `TextureHandle` bookkeeping is in CPU memory. On unified memory systems (Apple Silicon, integrated GPUs), the data IS in RSS.

**The UI problem:** The `(L:180 C:45)` parenthetical visually suggests these are sub-components of the 234 MB total. They're not. On discrete GPUs:
- `L` is a real subset of RSS
- `C` is a logical estimate of GPU allocation, not part of RSS at all
- The relationship `L + C ≈ total` does not hold

This explains why our test numbers showed an "unaccounted gap" — the gap isn't unaccounted, it's just that `C` was being mentally added to RSS when it shouldn't be on discrete GPUs.

**What we'd want instead:** A more honest display would split CPU memory from GPU memory:

```
Img: 12.3 FPS | RSS:872 GPU:340 (L:180 C:160)
```

Where `GPU:` comes from `device.get_internal_counters().hal.buffer_memory.read()` (real wgpu allocator memory, cross-platform). `L:` and `C:` would remain as logical cache contents for understanding cache behavior, but the headline numbers (`RSS:` and `GPU:`) would be honest measurements.

**Decision:** This is a UI clarity improvement, not a memory-usage fix. Out of scope for this branch. Tracked for a future change after the wgpu switch is finalized.

### Open question: settings exposure

The MemoryHints tuning is workload-dependent. The Manual { 64..128 MB } default is a conservative middle ground. Power users with abundant RAM may prefer Performance for marginally faster texture allocation; memory-constrained users may accept slower navigation for MemoryUsage savings. Plan: expose this as a setting in Preferences with three presets (Performance / Balanced / Low memory) requiring app restart to take effect.

## Key files

- `src/main.rs` — renderer CLI arg, `NativeOptions` with `dithering: false` and `wgpu_options`
- `src/app.rs` — `device.poll(Maintain::Wait)` in `update()` for frame pacing
- `Cargo.toml` — eframe features with both `glow` and `wgpu` enabled
- `devlogs/020_renderer_selection.md` — initial renderer selection implementation and findings
