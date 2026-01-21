# 075: Dataset V1 Frame Sampling

## Overview

Sampled frames from real robot recordings for segmentation annotation (dataset_v1).

## Source Videos

| Video | Resolution | FPS | Frames | Content |
|-------|------------|-----|--------|---------|
| camera_rec3.mp4 | 480x480 | 30 | 295 | Camera recording |
| grasp_demo4.mp4 | 480x480 | 20 | 510 | Grasp demonstration |

## Sampling Parameters

| Video | Sample Rate | Output Frames | JPEG Quality |
|-------|-------------|---------------|--------------|
| camera_rec3 | every 5 | 59 | `-qscale:v 2` |
| grasp_demo4 | every 10 | 51 | `-qscale:v 2` |
| **Total** | - | **110** | - |

### FFmpeg Commands

```bash
# camera_rec3: sample every 5th frame
ffmpeg -i camera_rec3.mp4 \
    -vf "select=not(mod(n\,5))" -vsync vfr \
    -qscale:v 2 \
    frame_%04d.jpg

# grasp_demo4: sample every 10th frame
ffmpeg -i grasp_demo4.mp4 \
    -vf "select=not(mod(n\,10))" -vsync vfr \
    -qscale:v 2 \
    frame_%04d.jpg
```

### JPEG Quality Note

- `-qscale:v 2`: Near-lossless quality (~16KB avg per frame)
- Default ffmpeg JPEG quality produces pixelated output (~4KB avg)
- Scale: 2-31, lower = better quality

## Output

- Location: `/home/gota/ggando/ml/salt/dataset_v1/images/`
- Naming: `camrec3_frame_XXXX.jpg`, `grasp4_frame_XXXX.jpg`
- Resolution: 480x480 (square, matches real inference pipeline)

## Annotation Workflow

```bash
cd /home/gota/ggando/ml/salt

# 1. Extract SAM embeddings
uv run --extra gpu python helpers/extract_embeddings.py \
    --checkpoint-path sam_vit_h_4b8939.pth \
    --dataset-path dataset_v1

# 2. Annotate with SALT
uv run --extra gpu python segment_anything_annotator.py \
    --dataset-path dataset_v1 \
    --categories "background,table,cube,static_finger,moving_finger" \
    --checkpoint-path sam_vit_h_4b8939.pth
```

## Next Steps

1. Annotate 110 frames in SALT
2. Merge with dataset_v0 annotations
3. Retrain EfficientViT seg model on combined dataset
