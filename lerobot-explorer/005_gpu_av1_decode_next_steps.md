# Devlog 005: GPU AV1 Decode — Findings & Path Forward

**Date:** 2026-03-15
**Branch:** `feat/gpu-video-decode` (formerly `feat/video-playback`)

## Key finding: the driver works

**vk-video crate's H.264 Vulkan Video decode produces real pixels on the same driver (590.48.01, RTX 3090).** Y range=[32,196] across 30 frames at 640×480. This definitively proves the NVIDIA Vulkan Video driver is functional. The grey output (Y=128) in our AV1 implementation is a code bug, not a driver issue.

```
$ cd /tmp/vkvideo_test && cargo run --release
Found adapter
Created device
H.264: 199394 bytes
Created decoder
Decoded 28 frames
  Frame 0: 640x480, 460800 bytes, range=[37,193]
  Frame 1: 640x480, 460800 bytes, range=[37,195]
  ...
  Frame 27: 640x480, 460800 bytes, range=[37,196]
Flushed 2 frames
```

## What's wrong with our AV1 decode

The AV1 decode pipeline is structurally complete — session creation, DPB management, inter frame references, decode commands all succeed. But every frame produces flat Y=128, UV=128 regardless of bitstream content, quantization values, or offset calculations.

The issue is in **how we construct the AV1-specific Vulkan structures** — specifically the sub-structures (`StdVideoAV1TileInfo`, `StdVideoAV1Quantization`, etc.) and/or the bitstream buffer format. Changing `base_q_idx` from 0 to 255 produces no change in output, meaning the GPU either ignores our sub-structures or can't parse the bitstream at the offsets we provide.

## Differences between vk-video's working H.264 and our AV1

| Aspect | vk-video H.264 (works) | Our AV1 (grey) |
|--------|----------------------|----------------|
| begin_coding reference_slots | ALL DPB slots (active + inactive with -1) | Only referenced slots (tried all slots too, no fix) |
| Image layout transitions | Within same command buffer, PipelineStageFlags2 | Separate pre-transition at init time |
| Destination image | Separate from DPB (VIDEO_DECODE_DST) | Same as DPB (coincide mode) |
| Bitstream upload | RBSP data (NAL payload, stripped header) | Various formats tried (full OBU, payload only) |
| DPB slot info in pNext | VideoDecodeH264DpbSlotInfoKHR via push_next | Raw p_next pointer (tried both) |
| Buffer size | Padded to min_bitstream_buffer_offset_alignment | Not padded |
| Query pool | Optional decode query pool | Not used |

## Recommended approach: extend vk-video with AV1

Rather than continuing to debug our custom Vulkan Video infrastructure, **fork vk-video and add AV1 codec support alongside its existing H.264**. This reuses vk-video's proven:

- Session/device creation and management
- DPB slot allocation with bitmap tracking
- Image layout transition patterns (PipelineStageFlags2)
- Command buffer pooling and recycling
- Bitstream buffer management with alignment padding
- Reference slot info construction (all slots for begin_coding)
- Frame sorting by picture order count

### What we bring from our AV1 work:

- **AV1 OBU parser** (`av1_obu.rs`, ~900 lines): sequence header, full frame header, tile info, quantization, loop filter, CDEF, restoration, global motion
- **AV1 reference management**: VBI→DPB mapping, refresh_frame_flags tracking, order hint management
- **AV1 decode parameter construction**: StdVideoDecodeAV1PictureInfo, StdVideoAV1SequenceHeader, all sub-structures

### Implementation plan:

1. Fork vk-video as a git dependency
2. Add AV1 OBU parser module (our existing `av1_obu.rs`)
3. Add AV1 decoder instructions (equivalent to vk-video's `decoder_instructions.rs` for H.264)
4. Add AV1 reference manager (our VBI→DPB mapping, adapted to vk-video's `ReferenceContext` pattern)
5. Add AV1 session parameters (StdVideoAV1SequenceHeader → VkVideoSessionParametersKHR)
6. Add AV1 decode command construction following vk-video's `do_decode()` pattern exactly
7. Test with the same H.264 test video re-encoded as AV1

The key advantage: vk-video's `do_decode()` handles all the Vulkan plumbing correctly. We only need to provide the AV1-specific structures. If the H.264 path works, the AV1 path using the same infrastructure should work too — any remaining issue would be purely in the AV1 structure values.

## Current state of the codebase

### What's on `main`:
- Full annotation tool MVP (dataset loading, video playback via software decode, annotation UI, export)
- Episode sliding window cache
- Video playback with bounded channel VideoPlayer
- All annotation features working

### What's on `feat/gpu-video-decode`:
- All of main's features
- GPU decode infrastructure (`src/gpu_decode/`):
  - `av1_obu.rs` (~900 lines): complete AV1 OBU bitstream parser
  - `vulkan_decoder.rs` (~1500 lines): AV1Decoder with Vulkan Video session, DPB, decode commands
  - `wgpu_device.rs` (~250 lines): shared Vulkan device (GpuSetup) for wgpu + video decode
  - `test_av1.rs` (~300 lines): integration tests
- GPU decode produces structurally correct decode (no crashes, 592 frames) but grey pixels
- Software decode fallback active in the app
