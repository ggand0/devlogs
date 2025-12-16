# RT-DETR Bounding Box Coordinate Fix - September 18, 2025

## Summary
Fixed critical bounding box coordinate mapping issue in RT-DETR implementation that was causing completely incorrect bbox positions in video output.

## Problem Description
RT-DETR inference was running without errors and generating detections, but all bounding boxes appeared in wrong locations (e.g., "person" labels on concrete surfaces instead of actual people). The issue persisted even after previous fixes to tensor dimension handling.

## Root Cause Analysis

### Initial Incorrect Assumption
The code was treating RT-DETR torch hub outputs as if they were already in original image coordinates (3840×2160), but this was wrong.

### Key Discovery
RT-DETR preprocessing resizes input images to 640×640, but the model outputs coordinates relative to this **processed image size**, not the original size. The `orig_target_sizes` parameter was incorrectly set to original image dimensions.

### Technical Details
- **Input**: Original video frames (3840×2160)
- **Processed**: Resized to 640×640 for model inference
- **Model Output**: Bounding boxes in (x1,y1,x2,y2) format relative to 640×640 image
- **Required**: Scale back to original 3840×2160 dimensions

## Fixes Applied

### 1. Corrected `orig_target_sizes` Parameter
**File**: `src/explore_cv/models/transformers/rtdetr.py:89-91`

**Before:**
```python
orig_target_sizes = torch.tensor([
    [self.orig_size[1], self.orig_size[0]]  # [height, width] from original image
]).to(self.device)
```

**After:**
```python
orig_target_sizes = torch.tensor([
    [self.processed_size[1], self.processed_size[0]]  # [height, width] from processed image
]).to(self.device)
```

### 2. Restored Coordinate Scaling
**File**: `src/explore_cv/models/transformers/rtdetr.py:129-144`

**Before (incorrect - no scaling):**
```python
# RT-DETR torch hub returns boxes in original image coordinates
# No scaling needed - use coordinates directly
x1, y1, x2, y2 = box
```

**After (correct - with scaling):**
```python
# RT-DETR torch hub returns boxes relative to processed image size (640x640)
# Scale them back to original image size
x1, y1, x2, y2 = box

# Scale coordinates from processed size to original size
x1 = x1 * scale_x
y1 = y1 * scale_y
x2 = x2 * scale_x
y2 = y2 * scale_y
```

## Scale Factor Calculations
- **Scale X**: `3840 / 640 = 6.0`
- **Scale Y**: `2160 / 640 = 3.375`

## Verification
- Tested with video: `sf_geary_081425_0_short.mp4`
- Command: `uv run explore-cv rtdetr ./data/sf_geary_081425_0_short.mp4 --output results/debug_scaling_fixed.mp4 --model-path rtdetr_r101vd --conf-threshold 0.3`
- Result: Bounding boxes now correctly positioned on actual objects (people, cars, etc.)

## Performance
- **Inference Time**: ~135ms per frame
- **FPS**: ~7.4
- **Accuracy**: Decent detection of people and vehicles with proper bbox positioning

## Key Lesson
Always verify coordinate system assumptions when working with object detection models, especially when preprocessing involves resizing. The model's internal coordinate system must match the postprocessing expectations.