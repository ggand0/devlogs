# Devlog 002: GPU AV1 Decode via Vulkan Video — In Progress

**Date:** 2026-03-15
**Branch:** `feat/video-playback`
**Status:** Key frame decode working, inter frames + app integration next

## Goal

Replace the current ffmpeg software AV1 decode path with true GPU hardware decode via Vulkan Video (`VK_KHR_video_decode_av1`). Decoded frames stay on the GPU as textures — zero CPU-side pixel handling, no GPU→CPU→GPU roundtrip.

## Why not ffmpeg hwaccel?

ffmpeg-next's hwaccel path still uses `av_hwframe_transfer_data()` to download decoded frames from the GPU to CPU RAM, then we'd re-upload to an egui `TextureHandle`. That's not real GPU decode — it's CPU decode offload with a memory roundtrip.

The proper path: Vulkan Video decodes AV1 directly on the GPU's fixed-function decoder. The output is a VkImage in NV12 format that can be imported into wgpu as a texture. Frames never touch CPU memory.

## Hardware

- **GPU:** NVIDIA GeForce RTX 3090 (GA102)
- **Driver:** 590.48.01
- **Vulkan:** 1.3.275
- **AV1 decode extension:** `VK_KHR_video_decode_av1` (confirmed present)
- **Video decode queue:** Family 3
- **Capabilities:** max 8192×8192, 16 DPB slots, 16 active references

## What's implemented

### 1. AV1 OBU Bitstream Parser (`src/gpu_decode/av1_obu.rs`, ~750 lines)

Parses AV1 Open Bitstream Units following the AV1 spec (aomediacodec.github.io/av1-spec/). The Vulkan AV1 decode API is "host-side parsed" — the application must extract header-level parameters from the bitstream and provide them to the GPU via Vulkan structures. The GPU handles transform, reconstruction, and loop filtering.

**Implemented:**

- **OBU header enumeration**: `parse_obu_headers()` scans a buffer and returns each OBU's type, byte offset, and size. Used to locate Sequence Header, Frame Header, and Tile Group OBUs in both codec extradata and per-packet data.

- **Sequence Header parser**: `parse_sequence_header()` extracts all fields needed by `StdVideoAV1SequenceHeader`:
  - Profile, dimensions, frame ID configuration
  - Tool enable flags (CDEF, restoration, superres, warped motion, etc.)
  - Order hint bits, screen content tools, integer MV
  - Color config (bit depth, chroma subsampling, color primaries/transfer/matrix)
  - Operating points with level and tier
  - Film grain presence flag

- **Frame Header parser**: `parse_frame_header()` extracts fields needed by `StdVideoDecodeAV1PictureInfo`:
  - Frame type (KEY, INTER, INTRA_ONLY, SWITCH)
  - show_frame, showable_frame, error_resilient_mode
  - refresh_frame_flags (which DPB slots to update)
  - order_hint, primary_ref_frame
  - Frame dimensions and superres
  - Reference frame indices (ref_frame_idx[0..7])
  - Note: quantization, loop filter, CDEF, segmentation params are left as defaults since the GPU parses these from the raw bitstream

- **BitReader**: Sub-byte field extraction supporting bits, leb128, uvlc (unsigned exp-Golomb), su (signed), byte alignment.

**Tested against real data:**
- Extradata from ffmpeg contains AV1CodecConfigurationRecord (4-byte header + OBUs). After skipping the 4-byte config header, the parser correctly finds and parses the Sequence Header OBU.
- Per-packet data contains Sequence Header + Frame OBUs.
- Validated: Profile 0, 640×480, 8-bit, order_hint_bits=7, CDEF disabled, film grain disabled.

### 2. Vulkan Video Capabilities Query (`src/gpu_decode/vulkan_decoder.rs`, ~120 lines)

Uses `ash` (Rust Vulkan bindings) to query the GPU's AV1 decode capabilities.

- **`check_av1_decode_support()`**: Iterates queue families to find one with `VK_QUEUE_VIDEO_DECODE_BIT_KHR`.

- **`query_av1_decode_caps()`**: Builds a `VkVideoProfileInfoKHR` with AV1 Main profile, 4:2:0 chroma, 8-bit depth. Calls `vkGetPhysicalDeviceVideoCapabilitiesKHR` to get:
  - Max/min coded extent
  - Max DPB slots and active reference pictures
  - AV1-specific max level IDC

**Test result on RTX 3090:**
```
Video decode queue family: 3
AV1 decode caps:
  max_coded_extent: 8192x8192
  max_dpb_slots: 16
  max_active_refs: 16
```

### 3. AV1Decoder Struct — Full GPU Resource Setup (DONE)

`AV1Decoder::new()` creates the entire Vulkan decode pipeline from a parsed sequence header:

```rust
pub struct AV1Decoder {
    // Vulkan handles
    _entry, instance, device, physical_device,
    video_decode_queue, video_queue_family,
    transfer_queue, transfer_queue_family,
    video_queue_fn: ash::khr::video_queue::Device,

    // Video session
    video_session: VkVideoSessionKHR,
    session_params: VkVideoSessionParametersKHR,
    session_memory: Vec<VkDeviceMemory>,

    // DPB (8 reference frame slots)
    dpb_images: Vec<VkImage>,          // NV12 format
    dpb_views: Vec<VkImageView>,
    dpb_memory: Vec<VkDeviceMemory>,
    dpb_slot_active: [bool; 8],

    // Output image (for display)
    output_image, output_view, output_memory,

    // Bitstream buffer (4MB, host-visible)
    bitstream_buffer, bitstream_memory,

    // Command pool on video decode queue
    command_pool,

    // Stream state
    sequence_header, width, height, frame_count,
}
```

**Session creation details:**
- `VkVideoSessionCreateInfoKHR` with AV1 Main profile, 4:2:0 chroma, 8-bit, max 8 DPB slots
- Session memory bound via `vkBindVideoSessionMemoryKHR` — the GPU reports N memory requirements, each allocated separately (different memory types for different internal purposes)
- Memory type search uses fallback: first tries DEVICE_LOCAL, then any matching type (NVIDIA's video session memory requirements don't always have DEVICE_LOCAL bit set)

**Session parameters:**
- `StdSequenceHeaderOwned` wrapper keeps `StdVideoAV1ColorConfig` and `StdVideoAV1TimingInfo` in `Box<>` so the raw C pointers in `StdVideoAV1SequenceHeader` remain valid
- `seq_header_to_std_owned()` translates our parsed OBU fields to the exact bitfield layout Vulkan expects (20 flag bits for SequenceHeaderFlags)

**DPB images:**
- 8 × NV12 images (`VK_FORMAT_G8_B8R8_2PLANE_420_UNORM`)
- Usage: `VIDEO_DECODE_DPB_KHR | VIDEO_DECODE_DST_KHR` (can serve as both reference and destination)
- Video profile list chained via `push_next` on `VkImageCreateInfo`

**Bitstream buffer:**
- 4MB host-visible, host-coherent buffer for uploading compressed AV1 data
- Usage: `VIDEO_DECODE_SRC_KHR`

**Tested:** `AV1Decoder::new()` succeeds on RTX 3090, all resources allocated.

### 4. Decode Command Recording (DONE — key frames)

`decode_frame()` implements the full Vulkan Video decode command sequence:

```
1. Parse OBUs → find Frame/FrameHeader OBU
2. Parse frame header → frame_type, refresh_frame_flags, ref indices, order_hint
3. Upload compressed bitstream → host-visible GPU buffer (memcpy)
4. Build StdVideoDecodeAV1PictureInfo:
   - Flags via setters (error_resilient, disable_cdf_update, use_superres, etc.)
   - frame_type, current_frame_id, OrderHint, primary_ref_frame
   - refresh_frame_flags, coded_denom, TxMode
   - Reference name → DPB slot mapping (referenceNameSlotIndices[7])
5. Build VkVideoDecodeAV1PictureInfoKHR:
   - frame_header_offset (byte offset in bitstream)
   - Chained as pNext on VkVideoDecodeInfoKHR
6. Record command buffer:
   cmd_begin_video_coding_khr(session, params, reference_slots)
   cmd_control_video_coding_khr(RESET) — for key frames only
   cmd_decode_video_khr(src_buffer, dst_picture, setup_slot, refs)
   cmd_end_video_coding_khr()
7. Submit to video decode queue, wait idle
8. Update DPB slot active bitmap from refresh_frame_flags
```

**Readback path** (CPU, for validation):
- `read_back_nv12()`: allocates staging buffer, copies Y plane (PLANE_0) and UV plane (PLANE_1) from DPB image, maps to host memory
- `nv12_to_rgba()`: BT.601 YCbCr→RGB conversion

**Test result:** 11,676 bytes AV1 key frame → GPU decode → 307,200 + 153,600 bytes NV12 → 1,228,800 bytes RGBA. 614,400 non-zero pixel bytes confirmed (50% — alpha channel is all 0xFF).

**Key implementation details:**
- `StdVideoDecodeAV1PictureInfoFlags` has 30 bitfield flags — used individual setters instead of `new_bitfield_1()` to avoid arg ordering bugs
- `VideoReferenceSlotInfoKHR` doesn't expose `p_picture_resource` as a builder method — constructed raw struct with `PhantomData` marker
- Memory type search needs fallback: NVIDIA's video session memory doesn't always have DEVICE_LOCAL bit, relaxed to "any matching type_bits"
- Tile info pointers set to null — the GPU parses tile offsets directly from the raw OBU bitstream in the compressed buffer

## Architecture plan (remaining)

### 5. Inter Frame Decode (NEXT)

For each frame:
1. Parse the Frame Header OBU to get frame type, refresh_frame_flags, order_hint, ref indices
2. Upload compressed bitstream data to the host-visible buffer
3. Fill `StdVideoDecodeAV1PictureInfo` from parsed header fields
4. Map AV1 reference frame names (LAST, GOLDEN, etc.) to DPB slot indices via `referenceNameSlotIndices[7]`
5. Fill `VkVideoDecodeAV1PictureInfoKHR` with frame header offset, tile count/offsets/sizes
6. Transition image layouts (DPB → VIDEO_DECODE_DPB, output → VIDEO_DECODE_DST)
7. Record command buffer:
   - `vkCmdBeginVideoCodingKHR` (bind session, DPB slots, reference frames)
   - `vkCmdControlVideoCodingKHR(RESET)` for key frames (reset DPB)
   - `vkCmdDecodeVideoKHR` (the actual decode)
   - `vkCmdEndVideoCodingKHR`
8. Submit to video decode queue with fence/semaphore synchronization
9. Wait for completion

Current key frame decode works but inter frames need proper DPB reference tracking — providing the correct reference slot indices and picture resources for each named reference (LAST, GOLDEN, ALTREF, etc.). The DPB slot allocation also needs to properly handle `refresh_frame_flags` to reuse slots.

### 6. Replace ffmpeg Software Decode in VideoPlayer

Wire `AV1Decoder` into the `VideoPlayer` background thread:
- Thread opens video with ffmpeg for demuxing only (extract AV1 packets)
- Each packet → `decoder.decode_frame(packet_data)`
- Decoded frame → NV12→RGBA → send via bounded channel
- Main thread displays as `TextureHandle` (same as current flow)

This replaces the CPU decode path while keeping the same `VideoPlayer` API.

### 7. wgpu Texture Export (zero-copy path, future)

Export the decoded NV12 VkImage to a `wgpu::Texture` via Vulkan external memory (`VK_KHR_external_memory`). Since both the Vulkan Video decoder and wgpu run on the same physical device, this is a lightweight handle export/import — no actual memory copy.

The NV12 texture then needs a YUV→RGB conversion shader (or use `VK_KHR_sampler_ycbcr_conversion`) before display in egui.

## Dependencies added

```toml
ash = "0.38"                # Vulkan bindings (includes VK_KHR_video_decode_av1 types)
gpu-allocator = "0.27"      # GPU memory allocation for DPB images
```

## Files

```
src/gpu_decode/
├── mod.rs                    # Module declarations
├── av1_obu.rs       ~750 ln  # AV1 OBU parser (sequence header, frame header, tile info)
├── vulkan_decoder.rs ~900    # AV1Decoder struct, session, DPB, decode, readback
└── test_av1.rs      ~260 ln  # Integration tests (OBU parse, caps query, decode)
                    ~1900 lines total in gpu_decode/
```

## Key technical notes

### AV1 OBU vs H.264 NAL

AV1 uses OBUs (Open Bitstream Units) instead of H.264's NAL units. The structure is similar — a type + size header followed by payload — but the field encoding is different (AV1 uses leb128 for sizes, H.264 uses start codes). The vk-video crate's H.264 parser pipeline (NALUSplitter → NALUParser → AccessUnitSplitter → DecoderInstructions) maps to AV1 as: OBUParser → TemporalUnitGrouper → DecoderInstructions.

### Host-side vs device-side parsing

Vulkan Video AV1 decode is "host-side parsed" — the application (CPU) parses the bitstream headers and provides structured parameters to the GPU. The GPU only handles the compute-heavy parts (inverse transform, motion compensation, loop filtering, CDEF, restoration, film grain synthesis). This is why we need the OBU parser even though the GPU does the actual decode.

### AV1 reference frame management

AV1 has 8 reference frame slots (Virtual Buffer Index). For each inter frame, `refresh_frame_flags` indicates which VBI slots to update with the decoded picture. The application maps VBI slots to Vulkan DPB slot indices via `referenceNameSlotIndices[]` in `VkVideoDecodeAV1PictureInfoKHR`. This mapping is simpler than H.264's complex adaptive/sliding-window marking.

### Film grain

The test video has film grain disabled. When enabled, Vulkan Video can apply film grain synthesis on the GPU. The `VkVideoDecodeAV1ProfileInfoKHR::filmGrainSupport` flag controls whether the session supports it. Decode output and reconstructed pictures (DPB) must use separate images when film grain is active.
