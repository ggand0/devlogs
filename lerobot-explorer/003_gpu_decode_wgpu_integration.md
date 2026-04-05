# Devlog 003: GPU AV1 Decode — wgpu Integration Required

**Date:** 2026-03-15
**Branch:** `feat/video-playback`
**Status:** GPU decode works in isolation, crashes when coexisting with eframe's Vulkan instance

## The problem

`AV1Decoder` creates its own Vulkan instance + device. eframe/wgpu also creates one for rendering. The RTX 3090's video decode queue (family 3) has **count=1** — two Vulkan instances cannot share it. The NVIDIA driver segfaults on the second `vkCmdDecodeVideoKHR` call.

Test (standalone Vulkan, no eframe): 30 frames decode correctly.
App (eframe + our Vulkan): segfault on frame 2's decode command.

## The fix

Access wgpu's underlying Vulkan device via `wgpu::hal::vulkan` API instead of creating a separate Vulkan instance. This means:

1. Get wgpu's `ash::Device`, `ash::Instance`, `VkPhysicalDevice` from eframe's render state
2. Create the AV1 video session on that existing device (add `VK_KHR_video_decode_queue` and `VK_KHR_video_decode_av1` extensions)
3. Decoded NV12 images are `VkImage` on the same device as wgpu — no export/import needed
4. Convert NV12→RGB via wgpu compute shader or `VK_KHR_sampler_ycbcr_conversion`
5. Display the resulting `wgpu::Texture` in egui directly

This eliminates both problems:
- No competing Vulkan instances (single device)
- No CPU readback (frames stay on GPU: decode → shader → display)

## wgpu HAL API

eframe exposes the wgpu render state via `cc.wgpu_render_state` in the `CreationContext`. From there:

```rust
let render_state = cc.wgpu_render_state.as_ref().unwrap();
let device = &render_state.device;        // wgpu::Device
let queue = &render_state.queue;          // wgpu::Queue
let adapter = &render_state.adapter;      // wgpu::Adapter

// Access raw Vulkan handles via HAL
unsafe {
    device.as_hal::<wgpu::hal::vulkan::Api, _, _>(|hal_device| {
        // hal_device: &wgpu::hal::vulkan::Device
        // Access: ash::Device, ash::Instance, VkPhysicalDevice
    });
}
```

The challenge: wgpu may not enable the video decode extensions by default. We may need to configure eframe to request them, or use `wgpu::DeviceDescriptor` with `required_features` that include the video extensions. If wgpu doesn't expose video extensions at all, we'd need to use `ash` to load the extension function pointers from the existing device.

## What works today

- `src/gpu_decode/av1_obu.rs` — AV1 OBU parser (sequence header, frame header)
- `src/gpu_decode/vulkan_decoder.rs` — Full AV1Decoder: session, DPB, decode, readback
- Unit tests decode 30 frames on RTX 3090 with correct pixel data
- `decode_all_frames_gpu()` in video.rs — full pipeline (demux → GPU decode → readback → RGBA)
- VideoPlayer currently uses software decode as fallback

## Files

```
src/gpu_decode/
├── mod.rs                    # Module declarations
├── av1_obu.rs       ~750 ln  # AV1 OBU parser
├── vulkan_decoder.rs ~1000   # AV1Decoder (standalone Vulkan — needs wgpu integration)
└── test_av1.rs      ~280 ln  # Tests (all pass in isolation)
```

## Implementation progress

### Shared device creation (DONE)

`GpuSetup::new()` creates a single Vulkan instance + device with both graphics (family 0) and video decode (family 3) queues. Wraps as wgpu via `Instance::from_raw` + `Adapter::device_from_raw`. Passed to eframe via `WgpuSetup::Existing`. App launches and renders correctly.

### AV1Decoder::from_shared (DONE)

Accepts existing `ash::Device` from `GpuSetup` instead of creating its own. Creates video session, DPB, bitstream buffer on the shared device. `owns_device: false` prevents destroying the device on drop.

### SharedGpuState global (DONE)

`OnceLock<Arc<SharedGpuState>>` stores the shared Vulkan handles so background decode threads can access them. Set once in `main()`, read by `decode_all_frames_gpu()`.

### Key frame decode on shared device: WORKS

Single key frames decode correctly on the shared device. `cmd_decode_video_khr` succeeds, readback produces correct NV12 pixel data. Tested in the full app with eframe rendering simultaneously.

### Inter frame decode on shared device: CRASHES

`cmd_decode_video_khr` segfaults during **command recording** (not submission) when reference slots are provided for inter frame decode. This is the same code that works in the standalone test.

**What was tried:**
- Shared device (from_shared): key frames OK, inter frames crash
- Video-only separate device (new_video_only): same crash — NVIDIA cannot have two VkDevices when either uses the video decode queue
- Mutex serialization around decode commands: no effect
- `device_wait_idle` before every decode: no effect
- Vulkan validation layers: no errors reported
- Pre-allocated fixed-size reference picture resource arrays (avoid Vec realloc)
- Fence-based sync instead of queue_wait_idle

**Key observations:**
- The crash is in `cmd_decode_video_khr` (command RECORDING, not submission)
- Frame 1 (KeyFrame, 0 refs, dst_slot=0) succeeds
- Frame 2 (InterFrame, 1 ref to slot 0, dst_slot=1) crashes
- ref_slot_indices = [0,0,0,0,0,0,0], reference_slots has 1 entry pointing to dpb_images[0]
- The standalone test with its own VkDevice decodes 30+ frames fine
- The difference: wgpu's `device_from_raw` wraps the same device

## Root cause: NVIDIA driver limitation

**Confirmed:** NVIDIA's driver cannot handle two VkInstances accessing the same GPU when either uses the video decode queue. Reproduced independently: running `vulkaninfo` concurrently with the standalone decode test causes the same SIGSEGV on frame 2. This is not a code bug — it's a driver/hardware limitation.

Implications:
- Cannot use a separate VkDevice for video decode alongside eframe's wgpu VkDevice
- Must use eframe's own VkDevice for video decode
- But wgpu's `device_from_raw` wrapper also crashes on inter frames (the wrapper may change device internal state)
- The shared device approach (GpuSetup) creates the VkDevice with both graphics and video extensions, wraps it for wgpu, then uses the same device for video decode — key frames work, inter frames crash

## Resolution: tile_sizes crash was the root cause all along

The entire multi-day investigation (multiple VkInstances, shared devices, thread safety,
DPB layout transitions) was chasing a red herring. The actual bug:

**NVIDIA's driver crashes when `p_tile_sizes` is non-null in `VkVideoDecodeAV1PictureInfoKHR`
for inter frames.** Key frames with `p_tile_sizes` work fine. Inter frames with it segfault
in `cmd_decode_video_khr`.

**The fix:** provide `tile_sizes` only for intra/key frames. Omit for inter frames.
The GPU parses tile layout from the raw AV1 bitstream for inter frames.

```rust
let mut av1_pic_info = VkVideoDecodeAV1PictureInfoKHR::default()
    .tile_offsets(&tile_offsets);  // always
if fh.is_intra_frame {
    av1_pic_info = av1_pic_info.tile_sizes(&tile_sizes);  // key frames only
}
```

**Current status:** App runs with GPU AV1 decode on the shared VkDevice (single instance,
GpuSetup creates device with both graphics + video decode queues, wraps as wgpu). All 592
frames of the test video decode without crash. Readback + NV12→RGBA produces frames but
colors are wrong (light grey instead of actual video). The NV12→RGBA conversion or the
readback byte layout needs fixing.

## Next approach: synchronous decode on main thread

Since the crash only occurs when two VkInstances exist simultaneously (even on different queue families), and wgpu automatically creates one, the decode must happen on the same device that wgpu uses, during a window where wgpu is not actively submitting work.

egui's `update()` callback runs between wgpu render passes. Decoding synchronously within `update()` would avoid concurrent Vulkan usage. This means:
- Decode one frame per `update()` call (on the main thread)
- No background decode thread needed
- Latency: ~3ms per frame (software decode) or <1ms (GPU decode) — both within the 33ms frame budget
- This serializes graphics rendering and video decode on the same VkDevice

## Current status (2026-03-15)

**Pipeline fully functional, pixel output still grey (Y=128).**

### What works:
- AV1 OBU parser: full sequence header + frame header parsing (quantization, loop filter, CDEF, restoration, segmentation, tile info)
- Vulkan Video session created on shared GpuSetup device (single VkInstance)
- DPB management with VBI→DPB mapping for inter frame references
- All 592 frames of test video decode without crash (30fps)
- GPU decode integrated into app via VideoPlayer with bounded channel
- Readback: NV12 copied from DPB to staging buffer via layout transition

### What doesn't work:
- Decoded pixels are all Y=128, UV=128 (flat grey) despite correct base_q_idx=134
- The TileInfo sub-structure likely has incorrect MI coordinate arrays
- NVIDIA driver crashes when p_tile_sizes is non-null for inter frames (workaround: provide tile_sizes only for intra frames)

### Root cause analysis:
The Y=128 grey output means the GPU's inverse quantization and reconstruction produce a flat neutral image. This happens when the sub-structure values (particularly TileInfo) don't match what's in the actual AV1 bitstream. The GPU uses the application-provided sub-structure values as "ground truth" — if they're wrong, the reconstructed image is meaningless even though the raw decode command succeeds.

### Key NVIDIA driver findings:
1. `p_tile_sizes` non-null on inter frames → segfault in cmd_decode_video_khr
2. All sub-structure pointers MUST be non-null when tile_offsets is provided
3. Multiple VkInstances in same process → video decode corruption (driver state leak)
4. frameHeaderOffset = absolute byte position of frame header in bitstream buffer
5. tile_offsets = absolute byte position of tile data in bitstream buffer

## Grey pixel output investigation (2026-03-15 continued)

### Symptom
All decoded frames produce Y=128, UV=128 (flat neutral grey) regardless of:
- base_q_idx value (tested 0, 134, 255 — output unchanged)
- Sub-structure values (zeroed vs parsed — output unchanged)
- frameHeaderOffset (0, 3, 13, 16 — output unchanged)
- Upload strategy (full packet vs Frame OBU only vs Frame OBU payload only)

### What this means
The GPU is NOT using our sub-structure values for pixel reconstruction.
It also doesn't seem to be parsing the tile data from the bitstream.
Y=128 is the default "neutral" output when the GPU can't find/decode
the actual compressed tile data at the provided tile_offsets position.

### What works
- Decode commands succeed for all 592 frames (no crashes)
- DPB reference tracking works correctly
- Readback produces consistent NV12 data (just all 128)
- The decode pipeline is structurally complete

### Confirmed via testing
- Setting base_q_idx=255 produces identical Y=128 — GPU ignores our value
- Removing all sub-structures (null) crashes when tile_offsets is present
- The GPU does USE tile_offsets (removing them changes output from 128 to 0)
- The GPU uses tile_sizes (providing them for inter frames crashes)

### Root cause hypothesis
The tile data offset or the bitstream format doesn't match what NVIDIA's
AV1 hardware decoder expects. The decoder receives the compressed data
but can't parse it at the position we indicate, so it falls back to
producing neutral grey. Need to research exactly what byte format
and offset conventions NVIDIA's AV1 Vulkan decoder expects.

### What needs to happen
1. Research how FFmpeg constructs the bitstream buffer for NVIDIA AV1 decode
2. Compare byte-for-byte against our buffer contents
3. Determine the exact meaning of tile_offsets in NVIDIA's implementation
   (relative to what? OBU start? Frame header start? Tile group header?)
4. Check if NVIDIA needs the tile group OBU header to be present or stripped

## Spec research findings (2026-03-15 late)

### Key finding from Vulkan spec + Khronos docs:

**`frameHeaderOffset` points to the OBU header byte** (including obu_type, extension_flag, has_size_field), NOT to the uncompressed header payload.

**`pTileOffsets` points to the compressed tile PAYLOAD data**, NOT to the tile group OBU header. Quote from spec: "Implementations only need the offsets and sizes of individual tiles but do not care about the grouping of tiles into tile group OBUs."

**The bitstream buffer must contain complete OBUs with their headers.** Not stripped payloads.

### What was wrong in our implementation:

We tried multiple buffer strategies:
1. Upload entire packet → frameHeaderOffset = OBU offset (13) → grey
2. Upload Frame OBU including header → frameHeaderOffset = 0 → grey  
3. Upload Frame OBU payload only (stripped header) → frameHeaderOffset = 0 → grey

The spec says #2 should work — buffer has complete OBU, frameHeaderOffset=0 points to OBU header.

But the tile_offsets were wrong in all cases. For a Frame OBU (type 6, combined frame header + tile group), the payload structure is:

```
[uncompressed_header data (N bits, byte-aligned)]
[tile_group_obu() data which contains:]
  [tile_start_and_end_present_flag (1 bit, only if TileCols*TileRows > 1)]
  [for single tile: tile compressed data starts HERE]
  [for multi tile: tile_size_minus_1 fields + compressed data]
```

Our frame header parser reports `header_bytes_consumed` which tells us where the uncompressed header ends. But the tile group data has its OWN structure — for a single-tile frame there's no tile group header, but the tile data offset might still be off by a few bits/bytes due to byte alignment.

### The fix needed:

For a Frame OBU uploaded to the buffer:
- `frameHeaderOffset` = 0 (OBU starts at buffer start)
- `tile_offsets[0]` = byte offset past the uncompressed header AND past any tile group framing, pointing to the actual tile compressed payload
- `tile_sizes[0]` = size of just the compressed tile data

The parser needs to account for the exact byte boundary between frame header and tile data, including any tile_start_and_end_present_flag bits and byte alignment padding.

### Sources:
- [VkVideoDecodeAV1PictureInfoKHR spec](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VkVideoDecodeAV1PictureInfoKHR.html)
- [VK_KHR_video_decode_av1 proposal](https://docs.vulkan.org/features/latest/features/proposals/VK_KHR_video_decode_av1.html)
