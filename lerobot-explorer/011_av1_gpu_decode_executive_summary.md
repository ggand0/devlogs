# AV1 GPU Video Decode: Executive Summary

## What it does

Decodes AV1 video on the GPU using Vulkan Video instead of CPU software decode.
The LeRobot dataset viewer plays AV1-encoded robot episode videos at full speed
using the GPU's dedicated hardware video decoder (NVDEC on NVIDIA).

## How it works

```
MP4 file
  -> ffmpeg demux (extract raw AV1 packets, CPU)
  -> OBU parser (parse AV1 bitstream headers, CPU)
  -> Vulkan Video decode (GPU hardware decoder)
  -> NV12 readback (GPU -> CPU staging buffer)
  -> NV12-to-RGBA conversion (CPU)
  -> egui texture upload (CPU -> GPU)
  -> display
```

### Key components

**OBU Parser** (`vk-video/src/parser/av1/obu.rs`)
Parses the AV1 Open Bitstream Unit format. Extracts SequenceHeader (stream config)
and FrameHeader (per-frame parameters like quantization, loop filter, CDEF, tile
layout). The parser must read every single bit correctly because the format is
bit-packed with variable-length fields. One missed bit shifts all subsequent values.

**Decoder Instructions** (`vk-video/src/parser/av1/decoder_instructions.rs`)
Converts parsed frame headers into Vulkan Video decode instructions. Handles
reference frame management (which previously decoded frames are used as references
for inter-frame prediction), show_existing_frame (re-display without decode), and
the bitstream packaging (what raw data the GPU needs).

**Vulkan Video Decoder** (`vk-video/src/vulkan_av1_decoder.rs`)
The Vulkan Video API interface. Creates a video session, manages the Decoded Picture
Buffer (DPB, an 8-layer GPU image storing reference frames), uploads compressed
bitstream data, and issues decode commands. The GPU's fixed-function video decoder
hardware does the actual decompression.

**App Integration** (`lerobot-explorer/src/video.rs :: gpu_decode_av1()`)
Wires everything together. Uses ffmpeg for demuxing only (extracting packets from
the MP4 container). Each packet is parsed, compiled into decode instructions, and
sent to the GPU. Decoded NV12 frames are read back and converted to RGBA for display.

### The DPB (Decoded Picture Buffer)

AV1 uses up to 8 reference frames. The DPB is an 8-layer Vulkan image on the GPU.
Each decoded frame is written to a DPB layer and can be referenced by future frames.
AV1 uses "Virtual Buffer Index" (VBI) slots that map to DPB layers. A key frame with
refresh_frame_flags=0xFF stores one decoded frame and makes all 8 VBI slots point to it.

## Cross-platform status

### Linux + NVIDIA: Working
- Tested on RTX 3090, driver 590.48.01
- Uses VK_KHR_video_decode_av1 extension
- NVIDIA supports AV1 hardware decode on RTX 30xx and newer

### Windows: Should work with changes
Vulkan Video works on Windows with NVIDIA drivers. The vk-video crate already
supports Windows (it's the platform Software Mansion primarily targets). The main
blocker is that our lerobot-explorer uses wgpu for rendering which creates its own
Vulkan device, separate from vk-video's device. On Windows this should still work
since NVIDIA's Windows Vulkan drivers support the same extensions.

**What would need testing:**
- VK_KHR_video_decode_av1 availability on Windows NVIDIA drivers
- The dual-device setup (wgpu for rendering, vk-video for decode)
- ffmpeg availability for demuxing

### macOS: Will NOT work as-is
macOS does not support Vulkan natively. It uses Metal. There is no VK_KHR_video_decode_av1
on macOS, even through MoltenVK (the Vulkan-to-Metal translation layer), because
MoltenVK does not implement Vulkan Video extensions.

**Options for macOS:**
1. **VideoToolbox** (Apple's native API): macOS has hardware AV1 decode on M3+ chips
   via VideoToolbox. Would need a completely separate decode backend using Apple's
   framework. This is how ffmpeg does it on macOS (`-hwaccel videotoolbox`).
2. **Software fallback**: The current libdav1d software decode path already works
   everywhere. It just uses CPU instead of GPU.
3. **ffmpeg hwaccel**: Use ffmpeg's hardware acceleration abstraction which handles
   VideoToolbox on macOS, VAAPI/NVDEC on Linux, DXVA2/D3D12 on Windows. This would
   replace our custom Vulkan Video path with a more portable solution, but loses
   the zero-copy GPU texture potential.

### Recommended cross-platform approach
Keep the current Vulkan Video path for Linux/Windows (where it works), and fall back
to software decode on macOS. The fallback is already implemented. Later, a
VideoToolbox backend could be added for macOS GPU decode if needed.

## Performance

- Standalone decoder: 592 frames in ~15s = ~40 fps (above 30fps target)
- Bottleneck: NV12-to-RGBA CPU conversion + GPU readback latency
- Future optimization: keep NV12 on GPU and convert via wgpu shader (zero-copy)

## What the vk-video fork contains

The AV1 support was added to a fork of Software Mansion's `smelter/vk-video` crate
(branch `feat/av1-decode` at `/home/gota/ggando/rust_gui/vk-video-fork/`).

Added files:
- `vk-video/src/parser/av1/` (obu.rs, mod.rs, reference_manager.rs, decoder_instructions.rs)
- `vk-video/src/vulkan_av1_decoder.rs`
- `vk-video/examples/decode_av1.rs`

Modified files:
- `vk-video/src/lib.rs` (module registration)
- `vk-video/src/parser.rs` (av1 module)
- `vk-video/src/device.rs` (AV1 extension enablement)
- `vk-video/src/vulkan_video.rs` (public exports)
