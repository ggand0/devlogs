# 009: AV1 Decode Complete — App Integration Issue

## Parser Status: COMPLETE
The AV1 OBU parser and Vulkan Video decoder are now fully working:
- **test_av1.ivf**: ALL 17/17 frames with real pixels
- **so101 full episode (592 frames)**: ALL 592/592 frames with real pixels
- Both key frames and inter frames decode correctly

## Root causes found and fixed (complete list)
1. `disable_frame_end_update_cdf` — read BEFORE tile_info when !reduced && !disable_cdf
2. `frame_refs_short_signaling` — read before ref_frame_idx for inter when enable_order_hint
3. `allow_intrabc` — conditional on allow_screen_content_tools (not always read)
4. `delta_q_present` flag — must be set in StdVideoDecodeAV1PictureInfoFlags
5. `lr_type` enum remap — AV1 spec and Vulkan use different enum orderings
6. `lr_uv_shift` — only read when chroma planes use loop restoration
7. CDEF strengths — must be passed through (were zeroed)
8. `LoopRestorationSize` — stores `1+lr_unit_shift` (not pixel sizes)
9. Loop filter condition — `!coded_lossless` not `!is_intra_frame`
10. `render_size` — only read for !error_resilient in inter path else branch

## Current issue: App playback still broken
The standalone `decode_av1` example produces 592/592 correct frames.
But the app (`video.rs::gpu_decode_av1`) shows flashes/jumps.

### Likely cause
The app's `gpu_decode_av1` function has issues:
1. Multiple threads: software decode (libdav1d) runs on cache/thumbnail threads
   simultaneously with GPU decode on the main video thread → visual conflicts
2. Frame indexing: the GPU path increments frame_count only on successful decode,
   but `show_existing_frame` frames produce no output, creating gaps
3. The first inter frame after the first key frame produces Y=0 (DPB init issue)
   which may cause the player to glitch

### What needs fixing
- The `gpu_decode_av1` in video.rs needs to handle ALL frames correctly
- show_existing_frame should re-output the referenced DPB frame
- Frame counter must match the expected frame count from the demuxer
- May need to suppress software decode threads when GPU decode is active
