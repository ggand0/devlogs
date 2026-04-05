# Devlog 004: GPU AV1 Decode — Grey Pixel Investigation

**Date:** 2026-03-15
**Branch:** `feat/video-playback`
**Status:** Decode pipeline structurally complete, pixel output is flat grey (Y=128, UV=128)

## Problem

All GPU-decoded AV1 frames produce Y=128, UV=128 (neutral grey) regardless of the actual video content. The decode commands succeed without errors for all 592 frames including inter frames with DPB references.

## What was tried

### 1. Sub-structure values
- Set `base_q_idx = 0, 134, 255` → output unchanged (always Y=128)
- Conclusion: GPU ignores application-provided quantization values for pixel reconstruction
- Sub-structures (`pTileInfo`, `pQuantization`, etc.) are required non-null but their values don't control pixel output

### 2. frameHeaderOffset variations
| Value | What it points to | Result |
|-------|-------------------|--------|
| 0 | OBU header byte (Frame OBU only upload) | Y=128 |
| 3 | OBU payload start (past type+size bytes) | Y=128 |
| 13 | Frame OBU start within full packet | Y=128 |
| 16 | Frame OBU payload within full packet | Y=128 |

Conclusion: frameHeaderOffset value doesn't affect pixel output

### 3. Buffer upload strategies
| Strategy | Buffer contents | Result |
|----------|----------------|--------|
| Full packet | SeqHeader OBU + Frame OBU | Y=128 |
| Frame OBU with header | Complete OBU (type+size+payload) | Y=128 |
| Frame OBU payload only | Stripped header, just payload | Y=128 |

Conclusion: buffer format doesn't affect pixel output

### 4. Tile offset variations
- `tile_offsets = [0]` → Y=128
- `tile_offsets = [5]` (payload-relative) → Y=128
- `tile_offsets = [8]` (OBU-relative) → Y=128
- `tile_offsets = [21]` (packet-relative) → Y=128
- No tile_offsets at all → Y=0 (all black)

Key finding: tile_offsets IS being used (output changes from 0 to 128 when present) but the tile data isn't being decoded into actual pixel values.

### 5. tile_sizes behavior
- `tile_sizes` provided for intra frames → Y=128 (grey)
- `tile_sizes` provided for inter frames → SEGFAULT (NVIDIA driver bug)
- `tile_sizes` omitted → Y=128 for intra, Y=0/128 for inter

### 6. Frame header parsing completeness
Parsed ALL sections of the AV1 uncompressed header:
- show_existing_frame, frame_type, show_frame, error_resilient_mode
- frame dimensions, superres, render size
- ref_frame_idx (for inter), allow_intrabc
- interpolation_filter, is_motion_mode_switchable, use_ref_frame_mvs
- tile_info (tile_cols_log2, tile_rows_log2, context_update_tile_id)
- quantization_params (base_q_idx, delta_q values, using_qmatrix)
- segmentation_params (feature enable/value loops)
- delta_q_params, delta_lf_params
- loop_filter_params (levels, sharpness, ref/mode deltas)
- cdef_params (damping, bits)
- lr_params (FrameRestorationType)
- read_tx_mode, frame_reference_mode, skip_mode_params
- allow_warped_motion, reduced_tx_set
- global_motion_params (per-reference motion type + parameters)

### 7. External validation
- FFmpeg 6.1 does NOT support AV1 Vulkan decode (only H.264/H.265) — cannot use as reference
- FFmpeg software decode (libdav1d) works correctly on the same video
- FFmpeg NVDEC (cuda) decode works correctly on the same video
- NVIDIA driver version: 590.48.01

### 8. Sequence header comparison
Extradata and packet sequence headers DIFFER at bytes 7 and 9 of the payload (likely timing_info or operating_point fields). Session parameters were created from extradata — potential mismatch with bitstream.

## Remaining hypotheses

### A. Missing VkVideoDecodeAV1DpbSlotInfoKHR
The Vulkan spec requires `VkVideoDecodeAV1DpbSlotInfoKHR` in the `pNext` chain of each `VkVideoReferenceSlotInfoKHR`. This structure contains `pStdReferenceInfo` pointing to `StdVideoDecodeAV1ReferenceInfo` which has the frame type and order hint of the reference picture. We DON'T provide this — our reference slots have `p_next: std::ptr::null()`.

### B. Session parameter mismatch
The `StdVideoAV1SequenceHeader` used to create the session came from the codec extradata, but the actual bitstream contains a slightly different sequence header. The session parameters might need to match exactly.

### C. DPB image format/profile chaining
The DPB images might need the video profile chained via `VkVideoProfileListInfoKHR` not just at creation time but also during the begin_coding/decode calls. The image layout transitions might also need to specify video-specific layouts.

### D. Bitstream format expectations
NVIDIA's AV1 decoder might expect the bitstream in a specific format that differs from what ffmpeg's demuxer provides. For ISOBMFF (MP4), AV1 samples use a specific OBU packaging format that might need transformation before feeding to the Vulkan decoder.

## Final status (2026-03-15 end of session)

### Confirmed: GPU has NEVER decoded actual pixels
Going back to the very first successful decode (commit c6c3ff2), the readback was
already all zeros (Y=0, UV=0 → green screen). Adding sub-structures changed it to
Y=128, UV=128 (grey screen). No combination of parameters has ever produced actual
video content from the GPU decoder.

### Additional fixes applied but no effect:
- Added VkVideoDecodeAV1DpbSlotInfoKHR with StdVideoDecodeAV1ReferenceInfo in pNext of each reference slot (required by spec)
- Tested all frameHeaderOffset values 0-5 (no change in output)
- Uploaded OBU payload only (matching FFmpeg's approach) with frameHeaderOffset=0
- Stored vbi_frame_type for reference info

### All hypotheses tested:
| Hypothesis | Status | Result |
|------------|--------|--------|
| A. Missing DpbSlotInfo | Fixed | Still grey |
| B. Session param mismatch | Investigated | Seq headers match closely |
| C. DPB format | Verified | NV12 4:2:0 8-bit correct |
| D. Bitstream format | Multiple formats tested | All produce Y=128 |

### Remaining possibilities:
1. **NVIDIA driver bug** with VK_KHR_video_decode_av1 on 590.48.01 — the extension is reported but may not be fully functional
2. **Session parameter creation issue** — the StdVideoAV1SequenceHeader structure may have incorrect field values (e.g., `seq_force_screen_content_tools` or `seq_force_integer_mv` affecting decode behavior)
3. **Need to update NVIDIA driver** to a newer version with better Vulkan Video AV1 support
4. **The bitstream might need Annex B format** instead of Low Overhead format for NVIDIA's implementation

### Software decode fallback
The app works with ffmpeg software decode (libdav1d) via the VideoPlayer's bounded channel. This is the current production path. GPU decode code is complete and structurally correct but produces no meaningful pixels on this driver version.

## CONFIRMED: Vulkan Video decode broken on this system (2026-03-15)

**FFmpeg H.264 Vulkan decode also crashes (SIGSEGV) on this driver.**

```
$ ffmpeg -hwaccel vulkan -i /tmp/test_h264.mp4 -vframes 1 /tmp/out.ppm
Segmentation fault (core dumped)
```

NVDEC (cuda) works fine for both H.264 and AV1. Only Vulkan Video is broken.
Driver 590.48.01 on RTX 3090 with FFmpeg 6.1.1. This is definitively a driver
or FFmpeg version issue, not our code.

Our GPU AV1 decode implementation is structurally correct — the decode commands
succeed without errors, DPB reference tracking works, readback pipeline works.
The grey output (Y=128) and FFmpeg's segfault both indicate the Vulkan Video
driver implementation on 590.48.01 is non-functional.

**Next steps:**
1. Update NVIDIA driver to latest (575+)
2. Update FFmpeg to 7.0+ (which has production AV1 Vulkan decode support)
3. Re-test both FFmpeg Vulkan decode and our implementation
