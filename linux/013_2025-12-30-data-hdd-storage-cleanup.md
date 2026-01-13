# /data_hdd Storage Cleanup

**Date:** 2025-12-30

## Overview

Cleaned up old project data on /data_hdd from previous work at:
- **Araya** (Japanese company, ML engineer, ~2022)
- **Perceptive** (American startup, 2019-2021)

## Araya Data Cleanup

### shimh Project (Parking Lot People/Vehicle Tracking)

Location: `/data_hdd/araya_data/-2022/shimh/`

**Kept:**
- `datasets/dataset_062822/` (15G) - Latest training dataset
- `datasets/eval_annotations_061622/` (14G) - Complete eval annotations (180 folders = 60 queries × 3 cameras)
- `raw_data/` (24G) - Edge case videos (kids, wheelchair users)

**Deleted:**
- `datasets/dataset_062022/` - Older dataset
- `datasets/dataset_062122/` - Older dataset
- `datasets/dataset_062322.zip`
- `datasets/dataset_062422.zip`
- `datasets/dataset_062522.zip`
- `datasets/dataset_062622.zip`
- `datasets/dataset_062722.zip`
- `datasets/merge_062022/`
- `datasets/merge_062122/`
- `datasets/merge_062322/`
- `datasets/merge_062422/`
- `datasets/merge_062522/`
- `datasets/merge_062622/`
- `datasets/merge_062722/`
- `datasets/eval_annotations_061622_png/` - PNG exports (JSON annotations in main folder sufficient)
- Various intermediate merge outputs

**Estimated savings:** ~727G

### Other -2022 Cleanup

**Deleted:**
- `data_eval051722/` (337G) - Superset of shimh eval data, redundant
- `eval_annotations/` (15G) - Incomplete set (31 folders vs 180 needed)
- `downloads_bak/`
- `eval_annotation_processed0/`

### shimh-synthdata (Unity Perception)

**Deleted:**
- `/data_hdd/araya_data/2023/shimh-synthdata/` (17G)
- `/data_hdd/araya_data/shimh-synthdata/` (12G)
- `/data_hdd/araya_projects/shimh-synthdata/` (218M)

**Estimated savings:** ~29G

### Araya Projects Kept

- `/data_hdd/araya_projects/shimh-pplcnt/` - People counting with ByteTrack
- `/data_hdd/araya_projects/shi-mh-human-detection/` - YOLOX detection training
- `/data_hdd/araya_projects/shi-welding-robot/`
- `/data_hdd/araya_projects/arc-welding-step2/`
- `/data_hdd/araya_projects/ajinomoto/`

## Perceptive Data Cleanup

Location: `/data_hdd/perceptive_data/`

### Kept

- `vamr/` (214G) - First delivered project, sentimental value
- `training_data/` (301G) - Rich annotations (33 CSV columns) for studying annotation methods
- `pa_training_data/MTL3/3_090220/validate/` (32G) - Sample validation data
- `perceptive_projects/perceptive-pipeline/` - Main project repository

### Deleted

- `pa_training_data/MTL3/` (except validate folder above) (~151G)
  - 1_082820/, 2_090120/, 3_090220/train/ - Less rich annotations (13 CSV columns, no image annotations)

### In Progress

- `binary_ar/walking_npy/` (214G) - Converting 7,885 .npy files to PNG format
  - Each .npy contains video frames as (N, 300, 300, 3) uint8 numpy arrays
  - PNG conversion running in background
  - Expected savings after conversion: ~110G

### Still to Review

- `ava/` (548G) - AVA action recognition dataset, mostly deletable
- `anaconda3/` (48G) - Old conda environment
- `tmp/` - Temporary files

## Programs Cleanup

**Deleted:** `/data_hdd/programs/` (84G)

Contents were:
- Unity Hub and old Unity versions (2020.3.18f1, 2021.2.0b6, 2021.3.6f1)
- Unreal Engine 5.1.0 (59G)
- Old Android SDK

All obsolete - current installations on main drive.

## Summary

| Category | Deleted | Kept |
|----------|---------|------|
| shimh datasets | ~727G | 53G |
| shimh eval data | 352G | - |
| shimh-synthdata | 29G | - |
| pa_training_data | 151G | 32G |
| programs | 84G | - |
| **Total** | **~1.3TB** | ~85G |

## Notes

- Verified eval_annotations_061622 matches QUERIES list in ppl_cnt_multicam.py source code
- walking_npy conversion will complete in ~5-6 hours, then original .npy files can be deleted
- ubuntu_gotahome_backups/ (724G duplicity backups) left untouched
