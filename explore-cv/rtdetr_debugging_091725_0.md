# RT-DETR Debugging Session - September 17, 2025

## Summary
Debugging RT-DETR inference issues with 4K video processing on AMD Radeon RX 7900 XTX GPU.

## Initial Problem
- **Error**: `RuntimeError: The size of tensor a (240) must match the size of tensor b (400) at non-singleton dimension 1`
- **Context**: Running RT-DETR on 4K video (3840x2160), model failing in positional embedding logic
- **Command**: `uv run explore-cv rtdetr ./data/sf_geary_081425_0_short.mp4 --output results/debug.mp4`

## Issues Fixed

### 1. Tensor Dimension Mismatch (RESOLVED)
**Problem**: Positional embeddings expecting different spatial dimensions than input
**Root Cause**: Dynamic aspect-ratio preserving resize causing inconsistent tensor sizes
**Solution**: Changed to fixed 640x640 input size in `rtdetr.py:71-79`

```python
# Before: Dynamic resize with aspect ratio preservation
# After: Fixed size resize
target_w, target_h = 640, 640
pil_image = pil_image.resize((target_w, target_h), Image.LANCZOS)
```

### 2. Wrong Output Format Handling (RESOLVED)
**Problem**: Code expecting dictionary output, but torch hub RT-DETR returns tuple
**Root Cause**: RT-DETR from torch hub returns `(labels, boxes, scores)` tuple, not dict
**Discovery**: Debug output showed `tuple` type with 3 elements
**Solution**: Added tuple handling in `postprocess()` method

```python
# Added handling for tuple format (labels, boxes, scores)
if isinstance(outputs, (tuple, list)) and len(outputs) >= 3:
    labels_tensor, boxes_tensor, scores_tensor = outputs[:3]
```

### 3. Batch Dimension Handling (RESOLVED)
**Problem**: Tensor unpacking error - "too many values to unpack (expected 4)"
**Root Cause**: Output tensors had batch dimension: `[1, 300, 4]` instead of `[300, 4]`
**Solution**: Added batch dimension handling

```python
# Handle batch dimension - take first batch
if labels_tensor.dim() > 1:
    labels_tensor = labels_tensor[0]
if boxes_tensor.dim() > 2:
    boxes_tensor = boxes_tensor[0]
if scores_tensor.dim() > 1:
    scores_tensor = scores_tensor[0]
```

## Current Status

### Model Performance
- **Inference Time**: ~120-135ms per frame
- **FPS**: ~7.4-8.3
- **Detection Count**: 300 candidates per frame
- **Score Range**: 0.16 - 0.41 observed

### Remaining Issues

#### 1. No Bounding Boxes Visible (UNRESOLVED)
- Model runs without errors
- Detections are being generated (confirmed via debug output)
- But no bounding boxes appear in output video
- **Hypothesis**: Issue in visualization/drawing logic

#### 2. GPU Hang (NEW ISSUE)
- **Symptom**: "HW Exception by GPU node-1 (Agent handle: 0x55cf74a379c0) reason :GPU Hang"
- **Impact**: System instability for 1-2 minutes
- **Context**: AMD Radeon RX 7900 XTX with ROCm
- **Timing**: Occurs after processing completes

## Next Steps
1. Investigate video drawing/visualization pipeline
2. Check confidence threshold filtering in practice
3. Address GPU stability issues
4. Verify coordinate transformation logic

## Technical Details
- **GPU**: AMD Radeon RX 7900 XTX
- **Framework**: PyTorch with ROCm
- **Model**: RT-DETR R50VD from torch hub
- **Input**: 4K video (3840x2160, 29.97 fps, 127 frames)
- **Processing**: Resize to 640x640, normalize with ImageNet stats