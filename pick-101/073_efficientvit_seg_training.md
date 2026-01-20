# 073: EfficientViT Segmentation Training

## Overview

Fine-tuning EfficientViT-B0 for 6-class semantic segmentation.

## Model Architecture

- **Backbone**: EfficientViT-B0 (timm `efficientvit_b0.r224_in1k`)
- **Decoder**: FPN-style with 64-channel lateral convs
- **Parameters**: ~0.85M total (0.7M backbone + 0.15M decoder)

## Training Results

| Resolution | Val IoU | Batch Size | Augmentations |
|------------|---------|------------|---------------|
| 84x84 | 0.817 | 16 | Flips only |
| 640x480 | 0.811 | 4 | Full (see below) |

### Data Augmentations (640x480)

- RandomResizedCrop (scale 0.7-1.0, ratio 0.9-1.1, 70% chance)
- Horizontal/vertical flips (50% each)
- Random rotation (±15°, 50% chance)
- ColorJitter (brightness/contrast/saturation ±30%, hue ±10%)
- Gaussian blur (30% chance)

## Dataset

| Split | Samples | Notes |
|-------|---------|-------|
| Train | 139 | 85% with augmentation |
| Val | 24 | 15% no augmentation |
| Excluded | 7 | Bad annotations |
| **Total** | 163 | Stratified by grasp/default |

### Class Mapping

| ID | Class | Description |
|----|-------|-------------|
| 0 | unlabeled | Pixels with no annotation |
| 1 | background | Arm, sky, everything else |
| 2 | table | Ground/floor surface |
| 3 | cube | Target object |
| 4 | static_finger | Fixed gripper finger |
| 5 | moving_finger | Actuated gripper finger |

### Rendering Order

Different rendering orders for grasp vs default frames:
- **Default**: bg -> table -> cube -> fingers (cube behind fingers)
- **Grasping**: bg -> table -> fingers -> cube (cube in front when held)

## Training Configuration

| Parameter | 84x84 | 640x480 |
|-----------|-------|---------|
| Epochs | 100 | 100 |
| Early Stopping | 20 epochs | 20 epochs |
| Batch Size | 16 | 4 |
| Learning Rate | 1e-4 | 1e-4 |
| Optimizer | AdamW | AdamW |
| Scheduler | CosineAnnealing (eta_min=1e-6) | CosineAnnealing (eta_min=1e-6) |
| Loss | Soft IoU | Soft IoU |

## Commands

### Generate Masks from COCO Annotations

```bash
uv run python scripts/generate_seg_masks.py \
    --dataset-path data/dataset_v0 \
    --output-dir data/dataset_v0/masks \
    --selections data/dataset_v0/selections.json \
    --excluded data/dataset_v0/excluded.json
```

### Train Model

```bash
# 640x480 (default)
uv run python src/training/train_efficientvit_seg.py \
    --dataset-path data/dataset_v0

# 84x84 (for drqv2 policy)
uv run python src/training/train_efficientvit_seg.py \
    --dataset-path data/dataset_v0 \
    --img-height 84 --img-width 84 --batch-size 16
```

### Inference

```bash
uv run python -m src.training.infer_efficientvit_seg \
    --checkpoint outputs/efficientvit_seg/best.ckpt \
    --input /path/to/video.mp4
```

### Options

```
--img-height N     Image height (default: 480)
--img-width N      Image width (default: 640)
--loss iou|ce      Loss function (default: iou)
--epochs N         Max epochs (default: 100)
--lr RATE          Learning rate (default: 1e-4)
--batch-size N     Batch size (default: 4)
--output-dir DIR   Checkpoint directory (default: outputs/efficientvit_seg)
```

## Output

- Best checkpoint: `outputs/efficientvit_seg/best.ckpt`
- Monitored metric: val/iou (maximize)

## Files

- `src/training/train_efficientvit_seg.py` - Training script
- `src/training/infer_efficientvit_seg.py` - Inference script (image/video/directory)
- `scripts/generate_seg_masks.py` - Mask generation from COCO
- `data/dataset_v0/` - Dataset with images, masks, annotations
