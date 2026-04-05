# 007: AV1 Vulkan Decode — Final 1-bit CDEF Drift

## Status (Updated)
AV1 Vulkan Video decode WORKS for videos with enable_restoration=false (test video
produces Y=[17,196] matching NVDEC). Videos with enable_restoration=true (so101 dataset)
still grey due to 1-bit CDEF parser drift that corrupts lr_type values.

Key fixes applied:
- Simple PictureInfo flags (error_resilient + force_integer_mv), NOT brute-forced NVIDIA flags
- LoopRestorationSize = [3,3,3] (log2 unit size, not pixel sizes like 256)
- TxMode=2 for key frames (TX_MODE_SELECT)

## What works
- StdVideoAV1SequenceHeader matches NVIDIA byte-for-byte
- Key frame qp, tile_off, tile_sz all match NVIDIA reference
- H.264 GPU decode works perfectly via vk-video BytesDecoder
- AV1 falls back to software decode (libdav1d) which works

## The remaining bug

### Symptom
All decoded AV1 frames produce Y=128 (grey) even when:
- Tile offsets match NVIDIA exactly (verified for both test videos)
- PictureInfo flags match NVIDIA (0x04080041 for key frames)
- TxMode forced to 2 (TX_MODE_SELECT, matching NVIDIA)
- base_q_idx matches (49 for so101, 57 for test video)
- loop_filter_level matches ([2,2,1,1] and [3,3,2,1])

### Root cause
The AV1 frame header parser (`obu.rs::parse_frame_header()`) has a 1-bit position
drift somewhere in the CDEF or loop_restoration section. The TOTAL header byte count
is correct (verified: 128 bits = 16 bytes for so101, matches NVIDIA), but INDIVIDUAL
field values after CDEF are shifted by 1 bit:

- Our parser reads `tx_mode_select` at bit 125 → value 0 → TxMode=1
- NVIDIA reads it at bit 124 → value 1 → TxMode=2
- This 1-bit difference comes from lr_params reading an extra bit

### Detailed bit trace (so101 key frame, 640x480)
```
Correct positions (from Python trace matching NVIDIA):
  pos 0-15:   basic header (show_existing, frame_type, show_frame, cdf, scr, fsize, oh, mystery)
  pos 16:     uniform_tile_spacing_flag = 1 (True)
  pos 17-18:  tile col/row increments (1x1 tiles)
  pos 19-26:  base_q_idx = 49
  pos 27-31:  delta_q reads
  pos 32:     segmentation_enabled = false
  pos 33:     delta_q_present = true
  pos 35:     delta_q_res
  pos 36:     delta_lf_present = false
  pos 36-63:  loop_filter (lf=[2,2,1,1], sharp=0, delta=false)
  pos 64-115: CDEF (damping=0, bits=2, 4 entries × 12 bits)
  pos 116-121: lr_type = [2, 0, 0]
  pos 122:    lr_unit_shift = 1
  pos 123:    lr_unit_extra_shift (since shift=1)
  pos 124:    lr_uv_shift
  pos 125:    tx_mode_select → bit=0 → TxMode=1  ← WRONG (NVIDIA gets 2)
  pos 126:    reduced_tx_set
  pos 127:    padding → byte_align → 128 bits = 16 bytes
```

NVIDIA gets TxMode=2 which means it reads tx_mode_select at bit 124 (value=1).
The 1-bit difference is in the CDEF or lr_params section.

### Possible sources of the 1-bit error
1. CDEF entry count: `cdef_bits=2` → 4 entries. If should be `cdef_bits=1` → 2 entries,
   that's 24 fewer bits. Too many to be a 1-bit error.
2. CDEF y_sec_strength encoding: The spec says `cdef_y_sec_strength = f(2)` but if
   the value is 4 (after mapping), there might be special handling we miss.
3. lr_unit_shift: We read shift=1, extra=1. If the bit at 122 is NOT lr_unit_shift
   but the previous CDEF entry's last bit was read wrong, everything shifts.
4. CDEF entry values: If one entry's `uv_sec_strength` reads 1 fewer bit due to
   a spec subtlety, that shifts lr_params by 1 bit.

### How to fix
1. Dump the CDEF sub-structure bytes from NVIDIA's decoder at decode time
   (add hex dump of `pCDEF` in VkVideoDecoder.cpp before CmdDecodeVideoKHR)
2. Compare with our CDEF values byte-by-byte
3. Find which CDEF field has the wrong value
4. Trace back to the exact bit position where the parser diverges
5. Fix the parser read

### Key files
- Parser: `/home/gota/ggando/rust_gui/vk-video-fork/vk-video/src/parser/av1/obu.rs`
  - `parse_frame_header()` — the CDEF section at ~line 1230
- Decoder: `/home/gota/ggando/rust_gui/vk-video-fork/vk-video/src/vulkan_av1_decoder.rs`
- Instructions: `/home/gota/ggando/rust_gui/vk-video-fork/vk-video/src/parser/av1/decoder_instructions.rs`
- NVIDIA ref: `/tmp/vk_video_samples/build/` (built, working, has debug prints)
  - Debug prints in `VkVideoDecoder.cpp:1353` dump PictureInfo before decode
  - Debug prints in `VulkanAV1Decoder.cpp:207` dump parser output

### Test files
- `/tmp/so101_test.ivf` — 5 frames from the real LeRobot AV1 dataset (640x480)
- `vk-video-fork/test_data/test_av1.ivf` — 30 frames, libaom-encoded (320x240)
- Both have matching tile offsets with NVIDIA but grey decode output

### Commits on feat/av1-decode
```
0c56b912 Add AV1 parser modules
28df3364 Enable VK_KHR_video_decode_av1 extension
21f5a756 Add AV1VulkanDecoder (key frame working with hardcoded values)
aac54b2e Fix inter frame parsing + multi-frame packets
c73a08ce Clean up debug code
fdfca147 Fix show_existing_frame handling
f058fa86 Parser fixes: correct frame_header_bytes, TxMode override
```
