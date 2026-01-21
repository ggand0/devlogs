# 076: Merged Dataset (v0 + v1.2)

## Overview

Merged sim dataset (v0) with real robot dataset (v1.2) for segmentation model training.

## Dataset Composition

| Source | Prefix | Count |
|--------|--------|-------|
| v0 (sim) | camera_rec0 | 23 |
| v0 (sim) | camera_rec1 | 9 |
| v0 (sim) | grasp_demo0 | 43 |
| v0 (sim) | grasp_demo1 | 47 |
| v0 (sim) | grasp_demo2 | 48 |
| **v0 subtotal** | | **170** |
| v1.2 (real) | camrec3_frame | 39 |
| v1.2 (real) | grasp4_frame | 24 |
| **v1.2 subtotal** | | **63** |
| **Total** | | **233** |

## Labels

- Grasping frames: 39 (26 from v0, 13 from v1.2)
- Excluded frames: 8 (from v0 only)

## Merge Script

```bash
uv run python scripts/merge_datasets.py
```

Output:
```
dataset_v0: 170 pairs copied
dataset_v1.2: 63 pairs copied

Total: 233 image/mask pairs
Images: 233
Masks: 233

v0 selections: 26 grasping frames
v1 grasped: 13 grasping frames
Merged selections: 39 total grasping frames
Excluded: 8 frames
```

## Directory Structure

```
data/dataset_merged/
├── images/           # 233 JPG images
├── masks/            # 233 PNG masks (class IDs 0-5)
├── selections.json   # 39 grasping frames
└── excluded.json     # 8 excluded frames
```

## Training Command

```bash
uv run python src/training/train_efficientvit_seg.py \
    --dataset-path data/dataset_merged \
    --selections data/dataset_merged/selections.json \
    --excluded data/dataset_merged/excluded.json \
    --output-dir outputs/efficientvit_seg_merged \
    --batch-size 4 \
    --epochs 100 \
    --lr 1e-4 \
    --loss iou
```

## Notes

- v0 images: 640x480 (sim renders)
- v1.2 images: 480x480 (real camera, square crop)
- Training script handles mixed resolutions via resize
