# /data SSD Storage Cleanup - 2026-01-04

## Summary
Cleaned up /data partition to free disk space. Reduced usage from heavy to 26% (224GB used, 647GB free).

## Major Cleanups

### Deleted
| Size | Path | Reason |
|------|------|--------|
| 119G | /data/ggando/video-gen/generative-models/checkpoints | SDXL 1.0 weights (outdated, 2023 model) |
| 46G | /data/work/apto/overhead_300_with_gt | Old dataset |
| ~88G | /data/work/araya/similarity_algo/results_* | Old experiment results (kept only results_ws0 with latest weights from Dec 2023) |
| ~15G | /data/work/araya_data/2022/japan_tower/data/res | 204k frames reduced to 102 samples |

### Moved to /data_hdd
| Size | Path | Reason |
|------|------|--------|
| 153G | /data/work/araya_data/2022/shimh | Old 2022 project data |

## Before/After

**Before:** Heavy usage (estimated 700G+ used)
**After:** 224GB used / 647GB free (26%)

## Notes
- Kept 102 sample images from japan_tower/data/res for project memory
- Kept results_ws0 in similarity_algo (contains latest weights from Dec 6, 2023)
- SDXL 1.0 checkpoints deleted - can re-download from HuggingFace if needed, or use newer models (FLUX, SD3.5)
