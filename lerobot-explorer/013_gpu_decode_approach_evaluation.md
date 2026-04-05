# GPU video decode approach evaluation

Date: 2026-03-29

## Question

Before merging the GPU decode feature, is vk-video the right choice for GPU
video decoding? What alternatives exist and which gives the best zero-copy path?

## Options evaluated

### 1. vk-video (current) — Software Mansion / smelter project

- **Rust ecosystem:** Actively maintained, funded company, published on crates.io
- **Codec support:** Upstream H.264 only. Our fork adds AV1.
- **Zero-copy potential:** **Already exists.** `WgpuTexturesDecoder` returns
  `wgpu::Texture` directly — decoded frames never leave GPU memory. We're
  currently using `BytesDecoder` (returns `Vec<u8>` NV12) which does the
  GPU→CPU readback internally. Switching to `WgpuTexturesDecoder` eliminates
  the round-trip entirely.
- **Platform:** Linux (NVIDIA, AMD), Windows. No macOS.
- **Complexity:** Low-medium — the API already exists, we just need to use it.

### 2. FFmpeg hwaccel via ffmpeg-next

- **Rust ecosystem:** `ffmpeg-next` is maintenance-only. Active fork is
  `ffmpeg-the-third`. Safe API doesn't cover hardware accel — requires
  unsafe FFI with `ffmpeg-sys-next` for `AVHWDeviceContext`, `get_buffer2`.
- **Codec support:** Everything FFmpeg supports.
- **Zero-copy potential:** Possible with `AV_HWDEVICE_TYPE_VULKAN` backend
  (provides own VkDevice, gets back `AVVkFrame` with VkImage handles), but
  requires extensive unsafe FFI and careful Vulkan extension negotiation.
- **Platform:** Best — NVDEC, VA-API, DXVA2, VideoToolbox.
- **Complexity:** Very high for zero-copy. Only worth it for macOS support
  or codecs beyond H.264/AV1.

### 3. VA-API via cros-libva

- **Rust ecosystem:** Google ChromeOS team maintains cros-libva + cros-codecs.
  Pre-1.0, sparse docs.
- **Codec support:** H.264, VP8/9, AV1, HEVC via cros-codecs parsers.
- **Zero-copy potential:** VA surfaces → DMA-BUF export → Vulkan import.
  But wgpu DMA-BUF import support is not yet stable (gfx-rs/wgpu#2320 open).
- **Platform:** Linux only.
- **Complexity:** High. DMA-BUF→Vulkan→wgpu path is uncharted in Rust.

### 4. NVDEC/CUVID via nvidia-video-codec-sdk

- **Rust ecosystem:** Low quality. Raw bindings exist, safe abstractions are
  all TODOs. 29 stars, unclear maintenance.
- **Zero-copy potential:** CUDA memory → Vulkan requires CUDA external memory
  interop (`VK_EXT_external_memory_fd`). Very complex and fragile.
- **Platform:** NVIDIA only.
- **Complexity:** Very high. Vulkan Video accesses the same NVIDIA decode
  hardware through a better-integrated API without CUDA interop.

### 5. GStreamer via gstreamer-rs

- **Rust ecosystem:** High quality bindings, well-maintained.
- **Codec support:** Everything. GStreamer 1.28 added Vulkan Video AV1/VP9.
- **Zero-copy potential:** Possible (VkImages from Vulkan decoders, DMA-BUFs
  from VA-API/NVDEC), but getting GStreamer buffers into wgpu textures has
  no existing Rust implementation.
- **Platform:** Best — everything including macOS via VideoToolbox.
- **Complexity:** High. Large dependency (~100MB+), pipeline architecture
  learning curve, extracting VkImage handles from internal buffers.

### 6. Direct Vulkan Video via ash

- **Rust ecosystem:** ash is excellent. Already a dependency.
- **Zero-copy potential:** Maximum control. Our `wgpu_device.rs` already
  creates a shared VkDevice. Can decode → VkImage → `texture_from_raw()`
  → wgpu::Texture directly.
- **Codec support:** Must implement each codec manually (~1500 lines for AV1).
- **Complexity:** Very high maintenance burden. vk-video exists to abstract this.

## Key finding: the problem is not vk-video, it's how we use it

vk-video provides two decoder APIs:
- `BytesDecoder` → returns `RawFrameData` (`Vec<u8>` NV12 pixels) — **this is what we use**
- `WgpuTexturesDecoder` → returns `wgpu::Texture` directly — **zero-copy, what we should use**

The GPU→CPU→GPU round-trip that makes our GPU decode 2x slower than software
is entirely caused by using `BytesDecoder`. The fix is to switch to
`WgpuTexturesDecoder`, which keeps decoded frames as GPU textures.

The remaining work to display NV12 textures in egui:
1. Write a WGSL shader (~30 lines) that samples Y plane (Plane0) and UV plane
   (Plane1) and outputs RGBA
2. Create a wgpu render pipeline for NV12 → RGBA blit on GPU
3. Register output RGBA texture with egui via `register_native_texture()`

wgpu now supports NV12 multi-planar textures as render attachments
(gfx-rs/wgpu PR #8307, merged).

## Conclusion

**vk-video is the right choice.** We just need to use its zero-copy API
(`WgpuTexturesDecoder`) instead of the readback API (`BytesDecoder`).

However, for the immediate goal of grid video playback at 480p, software
decode (312-681 fps) is more than sufficient. The zero-copy GPU path should
be implemented later when either:
- Resolution increases to 1080p+ where CPU decode becomes the bottleneck
- Grid playback with many concurrent streams needs it
- Zero-copy is needed for latency-sensitive use cases
