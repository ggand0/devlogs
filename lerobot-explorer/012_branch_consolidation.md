# Branch consolidation for GPU video decode

Date: 2026-03-29

## Context

The `feat/gpu-video-decode` (app) and `feat/av1-decode` (vk-video fork) branches
accumulated many debugging commits during AV1 Vulkan Video decode development.
Before merging to main and eventually open-sourcing, the history needed cleanup.

## What was done

### vk-video fork (`/home/gota/ggando/rust_gui/vk-video-fork`)

Consolidated 13 commits into 3 via interactive rebase with fixup:

| Before (13 commits) | After (3 commits) |
|---|---|
| Add AV1 parser modules | **Add AV1 parser modules for Vulkan Video decode** |
| Enable VK_KHR_video_decode_av1 extension | (squashed into above) |
| Add AV1 Vulkan Video decoder (key frame) | **Add AV1 Vulkan Video decoder (key frame decode working)** |
| Fix inter frame parsing (x3) | (squashed into above) |
| Clean up debug code | (squashed into above) |
| Fix flags, LoopRestoration, CDEF, delta_q | (squashed into above) |
| Fix frame_refs_short_signaling (x2) | **Fix AV1 inter frame reference handling** |
| Fix disable_frame_end_update_cdf | (squashed into above) |
| Fix VBI-to-DPB slot remapping | (squashed into above) |

Also removed `Co-Authored-By: Claude` footers from all 11 unpushed commits
via `git filter-branch --msg-filter`.

### lerobot-explorer app

Consolidated 30 commits (including 1 merge commit) into 4 linear commits.
Used a tree-diff approach since the merge commit made interactive rebase impractical:
created a clean branch from main and replayed diffs between key commit ranges.

| Before (30 commits) | After (4 commits) |
|---|---|
| 23 standalone AV1 decoder + driver investigation commits | **Add shared Vulkan device and AV1 decoder infrastructure** |
| Merge main + Wire vk-video H.264 | **Wire vk-video H.264 GPU decode into video player** |
| Add AV1 path + re-enable + rebuild | **Add AV1 GPU decode path with software fallback** |
| Fix streaming + VBI-to-DPB fix | **Fix AV1 app integration and inter frame references** |

Final tree verified identical to original (`git diff` = empty).

## Backups

Original branches preserved as:
- `archive/feat/av1-decode` (vk-video fork)
- `archive/feat/gpu-video-decode` (lerobot-explorer)

## GPU decode verification

Tested both codecs with software fallback replaced by `panic!()` to confirm
GPU path is actually taken (not silently falling back):

- **AV1** (`so101_pick_place_smolvla`, 640x480@30fps): played at 28fps, no panics
- **H.264** (`openarm-cube-organization-v2`, 640x480@30fps): played fine, no panics

Both confirmed using Vulkan Video GPU decode on NVIDIA driver 590.48.01.

## GPU vs SW decode benchmark

Added `--bench <video_file>` CLI flag to measure raw decode throughput
(unbounded channel, no UI backpressure). Results on NVIDIA RTX 3060:

```
AV1 (640x480, 592 frames, so101_pick_place_smolvla):
  SW (libdav1d):      312 fps
  GPU (Vulkan Video): 162 fps  ← 2x slower

H.264 (640x480, 20712 frames, openarm-cube-organization-v2):
  SW (libavcodec):    681 fps
  GPU (Vulkan Video): 313 fps  ← 2x slower
```

**GPU decode is slower than software at 480p.** The bottleneck is the CPU
readback path: GPU decode → vkCmdCopyImage to staging → map → memcpy →
nv12_to_rgba CPU conversion → egui texture upload. The Vulkan command
submission overhead and PCIe readback latency dominate at this resolution.

Software decode avoids all of this — the frame data never leaves main memory.

**When GPU decode would win:**
- Higher resolution (1080p+, 4K) where CPU decode becomes the bottleneck
- Zero-copy rendering: keep decoded frames as wgpu textures, render directly
  via a custom egui paint callback. Eliminates the GPU→CPU→GPU round-trip.
  This is the path that would make GPU decode actually faster.

**Decision:** Default to software decode for now. GPU decode infrastructure
is preserved for future zero-copy rendering. Prioritize implementing the
grid video playback feature first using SW decode to validate whether
concurrent multi-video 480p playback needs GPU acceleration at all.

## Next steps

- Create `dev` branch on vk-video fork, merge `feat/av1-decode` into it
- Merge `feat/gpu-video-decode` into app `main`
- Implement grid video playback (multiple concurrent episodes) with SW decode
  to determine if GPU acceleration is actually needed
- If SW decode can't handle concurrent grid playback, implement zero-copy
  GPU rendering (decoded frames stay as wgpu textures, no CPU readback)
- New feature branch for viewer-mode feature gating (see `docs/plans/002_viewer_feature_gating.md`)
