# 036 — PR #30: Configure wgpu memory hints via WgpuSetupCreateNew

**Author:** YelovSK (Patrik Hampel)
**Status:** Open, reviewed and ready to merge

## Summary

Replaces manual wgpu instance/adapter/device creation (`WgpuSetupExisting` + `pollster::block_on`) with egui's built-in `WgpuSetupCreateNew` callback. Removes the `pollster` dependency. 57 lines out, 24 in.

## The change

Before: `build_wgpu_setup()` manually created the wgpu instance, requested an adapter (with `compatible_surface: None`), then requested the device with custom memory hints. Passed the whole thing to eframe as `WgpuSetupExisting`.

After: `build_wgpu_setup()` returns a `WgpuSetupCreateNew` with a `device_descriptor` closure that receives the adapter and returns a `DeviceDescriptor` with the custom memory hints and limits. Eframe handles instance, surface, and adapter selection.

```rust
egui_wgpu::WgpuSetupCreateNew {
    device_descriptor: Arc::new(move |adapter| {
        let base_limits = if adapter.get_info().backend == wgpu::Backend::Gl {
            wgpu::Limits::downlevel_webgl2_defaults()
        } else {
            wgpu::Limits::default()
        };
        wgpu::DeviceDescriptor {
            label: Some("viewskater wgpu device"),
            required_features: wgpu::Features::default(),
            required_limits: wgpu::Limits {
                max_texture_dimension_2d: 8192,
                ..base_limits
            },
            memory_hints: memory_hints.clone(),
        }
    }),
    ..Default::default()
}
.into()
```

## Why merge this

The old `WgpuSetupExisting` approach was valid - it's a supported eframe API for passing pre-built wgpu resources. It worked fine on all tested platforms and never caused issues.

This PR is a simplification, not a bugfix:
- Fewer lines (57 -> 24)
- Removes `pollster` dependency
- Less code to maintain (no manual instance/adapter/device creation)
- Only specifies what we actually customize (limits, memory hints), delegates the rest to eframe

A minor side benefit: with `WgpuSetupCreateNew`, eframe does surface-aware adapter selection internally. The old code used `compatible_surface: None` which worked fine on desktop hardware but was technically a guess. Not a real problem in practice.

## Bug found: transparent window on launch (Linux)

### Root cause

The surface gets stuck in a permanent `Outdated` state after the app resizes the window on first frame.

Sequence:
1. eframe creates window at 1600x900 (logical 1280x720 * 1.25 scale), surface configured
2. App's first `update()` resizes window to physical 1280x720 (`initial_size_set` logic)
3. Surface reconfigured at 1280x720
4. `get_current_texture` returns `SurfaceError::Outdated` every frame
5. eframe's default `on_surface_error` returns `SkipFrame` for `Outdated`
6. Frame is skipped, surface never reconfigured, stuck in infinite skip loop
7. No frame ever presents - window is transparent

### Why `Outdated` happens

`Outdated` means the Vulkan swapchain is stale - the surface dimensions no longer match what the compositor expects. This occurs when the window is resized between `configure_surface()` and `get_current_texture()`. It's a race: sometimes the compositor acknowledges the resize before the next frame, sometimes it doesn't. On X11 with NVIDIA, the timing is non-deterministic.

### Why it's an eframe bug

eframe's default `on_surface_error` handler returns `SkipFrame` for `Outdated`. But skipping the frame without reconfiguring the surface means the next frame will also be `Outdated` - it never recovers. The correct response is `RecreateSurface` (which calls `configure_surface()` with the current dimensions).

### Fix

Override `on_surface_error` in `WgpuConfiguration` to return `RecreateSurface` for `Outdated`:

```rust
let wgpu_options = egui_wgpu::WgpuConfiguration {
    on_surface_error: Arc::new(|err| match err {
        wgpu::SurfaceError::Outdated => egui_wgpu::SurfaceErrorAction::RecreateSurface,
        _ => egui_wgpu::SurfaceErrorAction::SkipFrame,
    }),
    ..
};
```

### Fix verified

Added `on_surface_error` override in `WgpuConfiguration` (src/main.rs) to return `RecreateSurface` for `Outdated` instead of `SkipFrame`. With a `tracing::warn!` log to confirm it fires:

```
2026-06-05T22:30:13.167610Z  WARN viewskater_egui: wgpu: surface Outdated, recreating
```

The race condition still occurs (the WARN fired), but the surface recovers instead of entering the infinite skip loop. No transparent window.

FPS with fix: 60fps on 4K images without debug logging (normal). ~50fps with `RUST_LOG=viewskater=debug` due to per-frame string formatting/IO overhead, not the fix itself. The WARN fires once, not every frame.

### Why PR #30 avoids it

With `WgpuSetupCreateNew`, the surface is created later in eframe's lifecycle (after the window exists and is sized). The initial configure likely happens at the correct 1280x720 size from the start, so no resize race occurs. This is a timing side-effect, not a deliberate fix - the `on_surface_error` fix is the real solution.

## How WgpuSetupCreateNew works (eframe lifecycle)

When eframe receives `WgpuSetup::CreateNew`, it runs this sequence internally:
1. Window created
2. Surface created from window (`instance.create_surface(window)`)
3. Adapter selected with `compatible_surface: Some(&surface)`
4. Your `device_descriptor` closure called with the adapter
5. Device created from your returned `DeviceDescriptor`

## Defaults preserved

- `power_preference`: defaults to `HighPerformance` (with env override), same as old manual code
- Instance backends: same (`PRIMARY | GL`, with env override)
- GL limits fallback: still handled via adapter info check inside the closure

## WgpuSetupExisting vs WgpuSetupCreateNew

There is no official guidance in egui's docs or source saying one is preferred over the other. No deprecation, no warnings. Both are equally supported variants of the `WgpuSetup` enum. The doc for `WgpuSetupExisting` just says "Configuration for using an existing wgpu setup."

Use cases for `Existing`:
- Sharing a device with another subsystem
- Custom adapter selection logic beyond what `native_adapter_selector` provides
- Non-eframe event loops (like the iced version's custom winit loop)

Use cases for `CreateNew`:
- You only need to customize the DeviceDescriptor (limits, memory hints)
- You don't need control over instance/adapter creation
- Less code to maintain

## egui wgpu lifecycle (from source)

Documented in `Painter::new()` doc comment (`crates/egui-wgpu/src/winit.rs`):

> "Only the `wgpu::Instance` is initialized here. Device selection and the initialization
> of render + surface state is deferred until the painter is given its first window target
> via `set_window()`. (Ensuring that a device that's compatible with the native window is chosen)"

Order:
1. Instance - created in `Painter::new()` via `WgpuSetup::new_instance()`
2. Surface - created in `set_window()` via `instance.create_surface(window)`
3. Adapter - selected in `RenderState::create()` with `compatible_surface: Some(&surface)`
4. Device - `device_descriptor(&adapter)` closure called, then `adapter.request_device(&descriptor)`
5. Renderer - created with device and target format

Source files:
- `crates/egui-wgpu/src/winit.rs` - Painter::new(), set_window(), add_surface()
- `crates/egui-wgpu/src/lib.rs` - RenderState::create()
- `crates/egui-wgpu/src/setup.rs` - WgpuSetupCreateNew struct and new_instance()

## CreateNew vs Existing: when to use which

Default to `CreateNew`, reach for `Existing` only when you have a concrete reason.

### Use CreateNew when:

- Standard eframe app (egui owns the GPU context)
- You need custom features, limits, or memory hints (the `device_descriptor` closure covers this)
- You want egui to handle adapter selection and surface compatibility
- You need `adapter.limits()` to avoid crashes on constrained hardware (VMs, WSL, old GPUs)

Real-world examples:
- Rerun: custom adapter selector via `native_adapter_selector` - https://github.com/rerun-io/rerun/blob/a05a4f8bb018063e4a63b62036e447df0801fa28/crates/viewer/re_viewer/src/lib.rs
- Bitang: requests specific wgpu features for demoscene shaders - https://github.com/aedm/bitang/blob/33c467ff42a7ad8ae6b45d6b7b349ee587bd469e/crates/bitang/src/tool/runners/window_runner.rs
- Brush: requests adapter.features() + adapter.limits() for ML compute shaders - https://github.com/ArthurBrussee/brush/blob/6decd108d70f60af536bb804b9bc20f47ddcda6f/apps/brush-app/src/ui/mod.rs
- fractal-renderer: uses adapter.limits() to avoid crashes on constrained hardware - https://github.com/MatthewPeterKelly/fractal-renderer/blob/9c104603830cfc86377fe922b9e94f9a6e0f16c5/src/core/eframe_support.rs

### Use Existing when:

- Multiple subsystems share one device (e.g. path tracer + GUI on same device)
- You need Vulkan extensions egui doesn't expose (e.g. DMA-BUF for zero-copy video)
- You need to validate GPU capabilities before egui starts
- Background GPU init for faster startup
- Egui is a debug overlay on your own rendering engine

Real-world examples:
- vfx-rs: shares one device across path tracer, USD delegate, and OIDN denoiser - https://github.com/ssoj13/vfx-rs/blob/478bd8b6c02d4c3cfca1a0a8622ad17458d4c6ab/crates/view/vfx-view/src/gpu_ctx.rs
- iroh-live: needs DMA-BUF Vulkan extensions for zero-copy hardware decoder on Linux - https://github.com/n0-computer/iroh-live/blob/772bbdfbbb384f841ef788324fa9b736ca202caa/moq-media-egui/src/lib.rs
- SuiSuiView: prewarms GPU on background thread, hands off to egui via Existing - https://github.com/BK927/SuiSuiView/blob/44c5e0b18a3cc678950cebb36418a4a7b924d57f/src/app/handoff_preview.rs

### Viewskater conclusion

We only customize memory hints and limits. `CreateNew` with the `device_descriptor` closure is the right fit. We don't need `Existing` - we were only using it because `CreateNew` didn't have the `device_descriptor` closure when the code was first written (added in egui 0.31).

## References

- docs.rs: https://docs.rs/egui-wgpu/latest/egui_wgpu/struct.WgpuSetupCreateNew.html
- Usage example: https://github.com/emilk/egui/issues/7342 (same pattern: custom device_descriptor with MemoryHints)
- egui PR #5506: Extended WgpuSetup with device_descriptor closure and adapter selector
- egui-wgpu changelog: v0.30 added WgpuSetupExisting, v0.31 extended WgpuSetupCreateNew

### egui 0.31.1 source (wgpu setup lifecycle)

- eframe::run_native() entry: https://github.com/emilk/egui/blob/0.31.1/crates/eframe/src/native/run.rs
- Painter::new() + set_window(): https://github.com/emilk/egui/blob/0.31.1/crates/egui-wgpu/src/winit.rs
- RenderState::create() (adapter + device): https://github.com/emilk/egui/blob/0.31.1/crates/egui-wgpu/src/lib.rs
- WgpuSetupCreateNew struct: https://github.com/emilk/egui/blob/0.31.1/crates/egui-wgpu/src/setup.rs
