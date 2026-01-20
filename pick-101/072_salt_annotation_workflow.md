# 072: SALT Annotation Workflow

## Overview

Using SALT (Segment Anything Labelling Tool) to create training data for custom 5-class segmentation model.

## Prerequisites

Download SAM checkpoint:
```bash
cd /home/gota/ggando/ml/salt
wget https://dl.fbaipublicfiles.com/segment_anything/sam_vit_h_4b8939.pth
```

## Dataset Structure

```
/home/gota/ggando/ml/salt/dataset/
├── images/           # Real robot frames
├── embeddings/       # SAM embeddings (generated)
└── annotations.json  # COCO format output
```

## 1. Setup Dataset

```bash
cd /home/gota/ggando/ml/salt
mkdir -p dataset/images dataset/embeddings

# Copy sampled frames from pick-101
cp /home/gota/ggando/ml/pick-101/data/*_sampled/*.jpg dataset/images/
```

Current frame counts:
| Source | Sampled Frames |
|--------|----------------|
| grasp_demo0 | 51 |
| grasp_demo1 | 51 |
| grasp_demo2 | 51 |
| camera_rec0 | 27 |
| camera_rec1 | 12 |
| **Total** | **192** |

## 2. Extract Embeddings (GPU)

```bash
uv run --extra gpu python helpers/extract_embeddings.py \
    --checkpoint-path sam_vit_h_4b8939.pth \
    --dataset-path dataset
```

## 3. Generate ONNX Decoder

```bash
uv run --extra gpu python helpers/generate_onnx.py \
    --checkpoint-path sam_vit_h_4b8939.pth \
    --dataset-path dataset \
    --onnx-models-path models
```

## 4. Annotate

PyTorch mode (no ONNX needed):
```bash
cd /home/gota/ggando/ml/salt
uv run --extra gpu python segment_anything_annotator.py \
    --dataset-path dataset \
    --categories "background,table,cube,static_finger,moving_finger" \
    --checkpoint-path sam_vit_h_4b8939.pth
```

## 5-Class Segmentation Scheme

| Class ID | Name | Description |
|----------|------|-------------|
| 0 | background | Arm, sky, everything else |
| 1 | table | Ground/floor surface |
| 2 | cube | Target object |
| 3 | static_finger | Fixed gripper finger |
| 4 | moving_finger | Actuated gripper finger |

## Annotation Workflow (Per Image)

1. **Select category** in right side panel (radio button)
2. **Left-click** on object to add positive point
3. **Right-click** outside to add negative point (refine mask)
4. **Press 'n'** to ACCEPT the mask (adds to annotation list)
5. Repeat steps 1-4 for each object in the image
6. **Press 'd'** to go to next image
7. **Press Ctrl+S** periodically to save to file

**CRITICAL**: Clicking alone does NOT save. You MUST press 'n' to accept each mask!

## Keybindings

| Key | Action |
|-----|--------|
| Left click | Add positive point (inside object) |
| Right click | Add negative point (outside object) |
| **n** | **ACCEPT mask (required to save!)** |
| r | Reset/reject current mask |
| a / d | Prev / Next image |
| l / k | Increase / Decrease transparency |
| Ctrl+S | Save all accepted annotations to file |

## Output

`dataset/annotations.json` - COCO format annotations

## View Annotations

```bash
pip install coco-viewer
python -m cocoviewer -i dataset -a dataset/annotations.json
```

## Training Pipeline

After annotation:
1. Train lightweight segmentation model (e.g., MobileNetV3 + DeepLabV3)
2. Export to ONNX for real-time inference (~5ms target)
3. Combine with Depth Anything V2 Small for seg_depth observation
