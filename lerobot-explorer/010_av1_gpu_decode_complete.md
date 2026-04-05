# 010: AV1 Vulkan Video GPU Decode — Complete

## Result
592/592 frames decode with real pixels on the so101 LeRobot AV1 dataset.
Zero black, zero grey frames. Both H.264 and AV1 GPU decode working in the app.

## The three categories of bugs fixed

### 1. OBU Parser Bit Counting (the longest hunt)

The AV1 `uncompressed_header()` is a complex bit-level format where every section
must be fully consumed before the next. Missing even 1 bit shifts ALL subsequent
field values, causing the GPU to silently produce Y=128 grey.

**Bugs found and fixed:**

| Bug | Effect | Source |
|-----|--------|--------|
| Missing `disable_frame_end_update_cdf = f(1)` before tile_info | ALL frames shifted by 1 bit | Found via FFmpeg's `cbs_av1_syntax_template.c` |
| Missing `frame_refs_short_signaling = f(1)` for inter frames | Inter frames shifted by 1 bit | AV1 spec Section 5.9.2 |
| Loop filter condition `!is_intra_frame` instead of `!coded_lossless` | Key frames skipped loop filter parsing | AV1 spec comparison |
| Missing `render_size()` for intra frames | 1 bit missing for intra | AV1 spec Section 5.9.5 |
| `lr_uv_shift` read unconditionally | Extra bit when only Y uses LR | NVIDIA vk_video_samples comparison |
| `allow_intrabc` conditional on `allow_screen_content_tools` | Was reading unconditionally for intra | FFmpeg cbs_av1 reference |

**How they were found:** Byte-level comparison of `StdVideoDecodeAV1PictureInfo` between
our decoder and NVIDIA's `vk_video_samples` decoder. Each field mismatch was traced back
to a bit position error in the parser.

### 2. Vulkan Enum/Flag Mismatches

The Vulkan Video AV1 structs use different enum orderings and flag conventions
than the AV1 bitstream spec.

| Bug | AV1 Spec | Vulkan | Fix |
|-----|----------|--------|-----|
| `lr_type` enum | NONE=0, SWITCHABLE=1, WIENER=2, SGRPROJ=3 | NONE=0, WIENER=1, SGRPROJ=2, SWITCHABLE=3 | Remap table |
| `LoopRestorationSize` | Pixel sizes (64-256) | `1 + lr_unit_shift` (1-3) | Store shift+1 |
| Missing `delta_q_present` flag | Parsed but not set | `StdVideoDecodeAV1PictureInfoFlags::set_delta_q_present()` | Set from parsed value |
| CDEF strengths zeroed | Parser read them | Struct left at zero | Pass through all 4 arrays |
| `force_integer_mv` for key frames | 0 (no motion) | NVIDIA sets 1 | Force 1 for key frames |

### 3. DPB Slot Management (VBI→DPB Remapping)

**The bug:** AV1 uses 8 "Virtual Buffer Index" (VBI) slots that map to decoded
reference frames. When a key frame has `refresh_frame_flags=0xFF`, all 8 VBI slots
point to the same decoded frame. But that frame is stored in ONE DPB array layer
(e.g., layer 0). Our code was passing VBI indices (e.g., 2) as DPB slot indices
to `referenceNameSlotIndices`, causing the driver to read from empty DPB layers.

**The fix:** Remap VBI slot indices to actual DPB layer indices using the
`ref_id_to_slot` mapping before building the Vulkan decode structures.

**Evidence:** Frame 2 (first inter after first key) was consistently Y=0 (black)
while frame 4 (inter after second key) worked. Both had identical parsed values
(qp=120, tiles=1x1). The difference was the DPB slot mapping.

## Architecture

```
IVF/MP4 → ffmpeg demux → AV1 packets
  → parse_packet() → Vec<TemporalUnit>
  → compile_temporal_unit() → AV1DecoderInstruction
  → AV1VulkanDecoder::decode_frame()
    → Upload bitstream (full TU, frameHeaderOffset=0)
    → Build StdVideoDecodeAV1PictureInfo + sub-structures
    → Remap VBI→DPB for referenceNameSlotIndices
    → Remap lr_type AV1→Vulkan enum
    → cmd_begin_video_coding → cmd_decode_video → cmd_end_video_coding
    → read_back_nv12 → NV12→RGBA → egui
```

## Files
- `vk-video/src/parser/av1/obu.rs` — AV1 OBU parser (SequenceHeader, FrameHeader)
- `vk-video/src/parser/av1/decoder_instructions.rs` — compile_temporal_unit
- `vk-video/src/vulkan_av1_decoder.rs` — Vulkan Video decode pipeline
- `vk-video/examples/decode_av1.rs` — standalone test binary
- `lerobot-explorer/src/video.rs` — app integration (gpu_decode_av1)

## Key commits on feat/av1-decode
- `1c4264db` disable_frame_end_update_cdf fix
- `bab38365` frame_refs_short_signaling fix
- `00ce0684` delta_q_present + CDEF strengths + lr_type remap
- `3a9bc0ef` VBI→DPB slot remapping fix
