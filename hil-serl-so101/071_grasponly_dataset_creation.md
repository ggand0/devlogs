# 071: Grasp-Only Dataset Creation for Reward Classifier and HIL-SERL

## Overview

Created two datasets for the grasp-only HIL-SERL proof-of-concept:
1. **Reward classifier training dataset** - All episodes with frame-level success labels
2. **Offline HIL-SERL dataset** - Curated episodes for policy pretraining

## Source Data

Recorded two datasets using the randomized IK reset (devlog 070):

### Positive Dataset (`so101_grasp_only_v1`)
- 15 episodes, 991 frames total
- 8 second episode duration @ 10 fps
- All episodes successful grasps
- Repo: `gtgando/so101_grasp_only_v1`

### Negative Dataset (`so101_grasp_only_negative_v1`)
- 10 episodes, 936 frames total
- 10 second episode duration @ 10 fps
- Intentional failure demonstrations (not reaching/grasping cube)
- Episodes 02, 07, 08, 09 had partial success (accidentally grasped)
- Repo: `gtgando/so101_grasp_only_negative_v1`

## Annotation Format

Created annotation file `data/reward_classifier_grasponly_v1.txt`:

```
# dataset path: /home/gota/.cache/huggingface/lerobot/gtgando/so101_grasp_only_v1
00: 34-         # frames 34+ are success (1-indexed)
01: 46-
...
14: 40-48, 60-  # frames 40-48 success, then frames 60+ success

# /home/gota/.cache/huggingface/lerobot/gtgando/so101_grasp_only_negative_v1
00: all fail
01: all fail
02: 81-
...
09: 56-58,61-63,66-  # intermittent success ranges
```

Frame numbers are 1-indexed (matching extracted frame filenames from ffmpeg).

## Created Datasets

### 1. Reward Classifier Dataset

**Path:** `/home/gota/.cache/huggingface/lerobot/gtgando/so101_grasp_only_reward_v1`

**Contents:**
- 25 episodes (all from both source datasets)
- 1927 total frames
- 523 success frames (27.1%)
- Binary reward labels: 1.0 = success, 0.0 = failure

**Script:** `scripts/create_grasponly_reward_dataset.py`

**Labeling logic:**
- Parses annotation file to extract success frame ranges
- Converts 1-indexed annotations to 0-indexed parquet frame_index
- Sets `next.reward = 1.0` for success frames, `0.0` otherwise

### 2. Offline HIL-SERL Dataset

**Path:** `/home/gota/.cache/huggingface/lerobot/gtgando/so101_grasp_only_offline_v1`

**Contents:**
- 18 episodes total
  - Episodes 0-14: from positive dataset (all 15 successful grasps)
  - Episodes 15-17: from negative dataset (originally eps 02, 07, 08 - partial success)
- 1239 total frames

**Script:** `scripts/create_grasponly_offline_dataset.py`

**Episode remapping:**
| Output Episode | Source Dataset | Source Episode |
|----------------|----------------|----------------|
| 0-14 | so101_grasp_only_v1 | 0-14 |
| 15 | so101_grasp_only_negative_v1 | 2 |
| 16 | so101_grasp_only_negative_v1 | 7 |
| 17 | so101_grasp_only_negative_v1 | 8 |

## Verification

Both scripts include verification checks:

### Index Verification
- Global `index` field is contiguous (0 to N-1)
- `episode_index` correctly matches output episode number
- `frame_index` resets to 0 at start of each episode

### File Verification
- Parquet file count matches expected episodes
- Video file count matches expected episodes
- `meta/info.json` updated with correct totals
- `meta/episodes.jsonl` contains all episode metadata

### Output from merge script:
```
=== Verification ===
Parquet files: 18
Video files: 18
Global indices: OK (contiguous)
Episode indices: OK
Frame indices: OK
```

## Frame Extraction

Extracted all frames from source videos for annotation review:

```bash
# Frames saved to: <dataset>/frames/<episode>/frame_0001.jpg, frame_0002.jpg, ...
ffmpeg -i episode_XXXXXX.mp4 -q:v 2 frames/episode_XXXXXX/frame_%04d.jpg
```

## Usage

### Train Reward Classifier
```bash
python scripts/train_reward_classifier.py \
  --dataset gtgando/so101_grasp_only_reward_v1 \
  --output outputs/reward_classifier_grasp_only_v1
```

### HIL-SERL Training
Update `configs/grasp_only_hilserl_train_config.json`:
```json
{
  "dataset": {
    "repo_id": "gtgando/so101_grasp_only_offline_v1"
  },
  "env": {
    "reward_classifier_pretrained_path": "outputs/reward_classifier_grasp_only_v1/checkpoints/XXXX/pretrained_model"
  }
}
```

## Files Created

- `scripts/create_grasponly_reward_dataset.py` - Reward classifier dataset creation
- `scripts/create_grasponly_offline_dataset.py` - Offline dataset merging
- `data/reward_classifier_grasponly_v1.txt` - Frame-level success annotations
