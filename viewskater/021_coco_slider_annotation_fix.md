# COCO Annotation Slider Rendering Fix

**Date:** 2025-11-01
**Status:** ✅ Fixed

## Problem

COCO annotations (bounding boxes and segmentation masks) displayed at incorrect scale and position during slider navigation, even without zoom. Once the slider was released or during keyboard navigation, annotations displayed correctly.

### User Observation

> "For some images it looks correct and for others it's still wrong. I think if the image of slider rendering start is common image resolution like 640x480 it's working for images that have the same res during the slider movement or something"

This observation was the key clue: annotations were correct when the new image had the same dimensions as the previous image, indicating a **stale dimension** issue.

## Root Cause

**Race condition between image loading and UI rendering during slider movement.**

### The Issue

1. During slider movement, `SliderImageWidgetLoaded` message creates an `iced::widget::image::Handle` from image bytes and stores it in `pane.slider_image`
2. The Viewer widget measures this handle during rendering using `renderer.measure_image(&handle)` and gets **correct dimensions**
3. The UI code (in `src/ui.rs`) tried to get dimensions for BBoxShader by:
   - Looking up from the image cache (which hadn't been updated yet)
   - Falling back to `current_image` dimensions (which also hadn't been updated yet)
4. Result: BBoxShader received **stale dimensions from the previous image**

### Debug Evidence

From logs showing the mismatch:
```
Viewer draw: measured_image=(640,388), bounds=(960.0,557.2), final_size=(919.1,557.2)
BBox prepare: image=(427,640), bounds=(960,557.2), base_scale=0.871, center=(294.1,0.0)
```

The Viewer rendered a 640×388 image, but BBoxShader calculated positions using stale 427×640 dimensions from the previous image.

## Solution

**Extract and store image dimensions alongside the slider image handle.**

### Implementation

1. **Extract dimensions from image bytes** (`src/navigation_slider.rs:385-391`):
   ```rust
   let dimensions = match image::load_from_memory(&bytes) {
       Ok(img) => (img.width(), img.height()),
       Err(_) => return Err((pane_idx, pos)),
   };
   ```

2. **Pass dimensions with the handle** (`src/app.rs:99`):
   ```rust
   SliderImageWidgetLoaded(Result<(usize, usize, Handle, (u32, u32)), (usize, usize)>),
   ```

3. **Store dimensions in Pane** (`src/pane.rs:57`):
   ```rust
   pub slider_image_dimensions: Option<(u32, u32)>,
   ```

4. **Use stored dimensions during slider movement** (`src/ui.rs:340-353`):
   ```rust
   let image_size = if app.is_slider_moving && app.panes[0].slider_image.is_some() {
       if let Some(dims) = app.panes[0].slider_image_dimensions {
           dims  // Use dimensions extracted when handle was created
       } else {
           // Fallback to current_image
       }
   } else {
       // Normal mode: lookup from cache
   }
   ```

## Why This Works

Both the Viewer widget and BBoxShader now use dimensions extracted from the **exact same image bytes** that created the `slider_image` handle. This eliminates the race condition:

- **Before**: Viewer measures from handle ✓ | BBoxShader uses stale cache ✗
- **After**: Viewer measures from handle ✓ | BBoxShader uses dimensions from same bytes ✓

## Files Modified

- `src/app.rs`: Updated `SliderImageWidgetLoaded` message and handler
- `src/pane.rs`: Added `slider_image_dimensions` field
- `src/ui.rs`: Use slider dimensions during slider movement
- `src/navigation_slider.rs`: Extract dimensions when creating handle

## Testing

Verified that annotations now render correctly during slider movement across images with different dimensions. The position and scale match the Viewer widget's rendering perfectly.
