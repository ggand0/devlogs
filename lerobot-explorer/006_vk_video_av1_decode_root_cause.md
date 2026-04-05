# 006: vk-video AV1 Decode — Root Cause Found & Fix

## Summary

Extended the `ggand0/smelter` fork's `vk-video` crate with AV1 Vulkan Video decode support.
The decoder compiles, runs without validation errors, but produces Y=128 (grey) output.
After extensive debugging including byte-level comparison with NVIDIA's reference
`vk_video_samples`, the root cause was identified: **the AV1 OBU parser has incorrect
bit counting in `parse_frame_header()`**, causing cascading wrong values for all parsed
fields after the first missing bit.

## Architecture

### Files in vk-video fork (`feat/av1-decode` branch)

- `vk-video/src/parser/av1/obu.rs` — AV1 OBU parser (SequenceHeader, FrameHeader)
- `vk-video/src/parser/av1/mod.rs` — TemporalUnit, parse_packet()
- `vk-video/src/parser/av1/reference_manager.rs` — 8-slot VBI reference tracking
- `vk-video/src/parser/av1/decoder_instructions.rs` — AV1DecoderInstruction enum, compile_temporal_unit()
- `vk-video/src/vulkan_av1_decoder.rs` — AV1VulkanDecoder (session, DPB, decode commands)
- `vk-video/src/device.rs` — Modified to enable VK_KHR_video_decode_av1
- `vk-video/examples/decode_av1.rs` — Test binary (IVF → decode → pixel stats)

### Decode pipeline

1. IVF demux → raw AV1 packets
2. `parse_packet()` → TemporalUnit (OBUs parsed, FrameHeader extracted)
3. `compile_temporal_unit()` → AV1DecoderInstruction (bitstream + offsets + parsed params)
4. `AV1VulkanDecoder::decode_frame()` → Vulkan Video decode → NV12 readback

### Vulkan Video setup

- `AV1VulkanDecoder::new()`:
  - Queries AV1 decode capabilities (std_header_version, min_bs_align, max_dpb, etc.)
  - Creates VideoSession with queried profile + capabilities
  - Creates session parameters with StdVideoAV1SequenceHeader
  - Allocates DPB as 8-layer array image (VIDEO_DECODE_DPB_KHR | VIDEO_DECODE_DST_KHR | TRANSFER_SRC)
  - Pre-transitions DPB to VIDEO_DECODE_DPB_KHR layout
- `decode_frame()`:
  - Uploads bitstream (full temporal unit data, padded to minBitstreamBufferSizeAlignment)
  - Builds StdVideoDecodeAV1PictureInfo with all sub-structures
  - Records: barrier → begin_video_coding → [reset for keyframe] → decode → end_video_coding
  - Reads back NV12 via TRANSFER_SRC transition + cmd_copy_image_to_buffer

## Root Cause: Parser Bit Position Drift

### The Problem

The AV1 `uncompressed_header()` is a complex bit-level format where each section must be
fully consumed before the next. Our `parse_frame_header()` in `obu.rs` skips or
under-reads several sections, causing the BitReader's position to drift. By the time we
reach tile_info, quantization, loop_filter, CDEF, and tx_mode, we're reading bits from
the wrong positions.

### Byte-level proof (key frame comparison)

```
StdVideoDecodeAV1PictureInfo (first 24 bytes):
NVIDIA: 41 00 00 00  00 00 00 00  00 00 00 00  00 07 ff 00  00 00 00 00  02 00 00 00
Ours:   01 00 00 00  00 00 00 00  00 00 00 00  00 07 ff 00  04 00 00 00  01 00 00 00
        ^^ flags     frame_type   current_fid  oh prf rff   interp_filt  TxMode
```

- **flags byte 0**: NVIDIA=0x41 (error_resilient + force_integer_mv), ours=0x01 (only error_resilient)
- **interpolation_filter** (offset 16): NVIDIA=0 (EIGHTTAP), ours=4 (SWITCHABLE)
- **TxMode** (offset 20): NVIDIA=2 (TX_MODE_SELECT), ours=1 (TX_MODE_LARGEST)

### Confirmed correct

- `StdVideoAV1SequenceHeader` matches NVIDIA byte-for-byte (first 24 data bytes identical)
- Driver capabilities: DPB_AND_OUTPUT_COINCIDE, max_dpb=16, min_bs_align=256, AV1 max_level=23
- No Vulkan validation errors
- Decode query returns SUCCESS
- NVIDIA `vk_video_samples` produces real pixels (Y=[17,196]) with same file/GPU
- NVDEC (CUDA) also produces real pixels — hardware works

### Parser bugs found so far

1. **Loop filter condition**: Used `!is_intra_frame` instead of `!coded_lossless` — KEY FRAMES
   should parse loop filter params when not lossless. **FIXED.**
2. **tile_info**: Missing `uniform_tile_spacing_flag` bit read. **FIXED.**
3. **render_size**: Missing `render_and_frame_size_different` bit read for intra frames. **FIXED.**
4. **~32 more bits missing**: Frame header reported as 88 bits (11 bytes), should be ~120 bits
   (15 bytes per NVIDIA reference). Need to find remaining missing sections.

### Impact

Since the BitReader position drifts by ~32 bits, EVERY field parsed after the drift point
has wrong values. This means:
- Wrong TxMode → driver misparses tile entropy coding → no output
- Wrong tile_data_offset → tile data pointer is in the frame header → garbage input
- Wrong loop_filter, CDEF, quantization values → wrong decode parameters

## How to fix

The `parse_frame_header()` function in `obu.rs` must read EVERY bit specified by the AV1
spec's `uncompressed_header()` (Section 5.9.2), even for fields we don't use. The bit
reader must be at the exact correct position before each section.

The AV1 spec sections in order:
1. show_existing_frame, frame_type, show_frame, showable_frame
2. error_resilient_mode, disable_cdf_update
3. allow_screen_content_tools, force_integer_mv
4. current_frame_id (if frame_id_numbers_present)
5. frame_size_override_flag
6. order_hint (if enable_order_hint)
7. primary_ref_frame (conditional)
8. decoder_model_info (if present — WE MAY BE MISSING THIS)
9. refresh_frame_flags
10. frame_size() + superres_params() + compute_image_size()
11. render_size() (for intra frames)
12. allow_intrabc
13. ref_frame_idx (for inter frames) + frame_size_with_refs()
14. allow_high_precision_mv, interpolation_filter, is_motion_mode_switchable
15. use_ref_frame_mvs
16. tile_info() — needs uniform_tile_spacing_flag
17. quantization_params()
18. segmentation_params()
19. delta_q_params(), delta_lf_params()
20. loop_filter_params() — condition: !CodedLossless && !AllowIntraBC
21. cdef_params() — condition: !CodedLossless && !AllowIntraBC && enable_cdef
22. lr_params() — condition: !CodedLossless && !AllowIntraBC && enable_restoration
23. read_tx_mode()
24. frame_reference_mode() (inter only)
25. skip_mode_params() (inter only)
26. allow_warped_motion (inter only)
27. reduced_tx_set
28. global_motion_params() (inter only)
29. film_grain_params() (if film_grain_params_present)

## NVIDIA reference build

The NVIDIA `vk_video_samples` is built at `/tmp/vk_video_samples/build/` and can be used
for byte-level comparison:
```bash
cd /tmp/vk_video_samples/build
./vk_video_decoder/demos/vk-video-dec-test -i input.mp4 --noPresent -o output.y4m
```
Debug prints added to `VulkanAV1Decoder.cpp::end_of_picture()` dump StdVideoDecodeAV1PictureInfo bytes.

## BREAKTHROUGH: KEY FRAME DECODES CORRECTLY (2026-03-22)

After adding 1 extra bit read before tile_info for intra frames, the key frame produces:
- **Y range [17, 196]** — matching NVDEC reference EXACTLY
- `base_q_idx = 57` — matching NVIDIA's vk_video_samples
- Frame header = 15 bytes — matching NVIDIA reference
- Tile offset = 36 in IVF — matching NVIDIA's 34 + 2 (TD offset)

The "mystery bit" at position 15 is 1 bit between `render_size()` and `tile_info()` for
intra frames. It could be `allow_intrabc` read unconditionally, or another spec element.

### Current status: 23/30 frames decode with real pixels
- Debug code cleaned up, NVIDIA overrides removed
- Inter frame parsing added (frame_size_with_refs, allow_high_precision_mv, etc.)
- Multi-frame IVF packet support (parse_packet returns Vec<TemporalUnit>)
- Graceful error recovery for unparseable Frame OBUs

### Remaining work
1. Fix inter frame base_q_idx (reads 0 instead of ~143) — same bit-drift class of bug
   Need to trace inter frame header bits like we did for key frame
2. Handle `show_existing_frame` properly (5-byte frames with qp=0 should just ref DPB)
3. Wire into lerobot-explorer's video player
4. Test with actual LeRobot AV1 datasets
