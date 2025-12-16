# Slider FPS Tracking Fix

**Date:** 2025-11-04
**Status:** Fixed

## Problem

The slider FPS display was showing inflated values (~30 FPS when experiencing ~5 FPS).

### Root Causes

1. **Wrong FPS calculation formula**: Used `(len() - 1) / elapsed` instead of `len() / elapsed`
2. **Wrong tracking point**: Tracked ALL async image loads (including stale/preloaded images), not just displayed images
3. **Async load inflation**: When slider moves quickly, multiple images load in parallel but only one is displayed

The key issue: we tracked every `SliderImageWidgetLoaded` message, but during rapid slider movement, many of those are for stale positions that are never displayed.

## Solution

### 1. Fixed FPS Calculation

**Before:**
```rust
return (self.frame_timestamps.len() - 1) as f64 / time_span;
```

**After:**
```rust
return self.frame_timestamps.len() as f64 / elapsed;
```

This matches the existing image FPS calculation in `navigation_keyboard.rs`.

### 2. Track Only Displayed Images

**Before:** Tracked every `SliderImageWidgetLoaded` message (including stale/preloaded images)

**After:** Only track when `pos == current_slider_pos`

```rust
// Get current slider position before borrowing pane mutably
let current_slider_pos = if pane_idx == 0 {
    app.slider_value as usize
} else if let Some(pane) = app.panes.get(pane_idx) {
    pane.slider_value as usize
} else {
    pos // Fallback if pane not found
};

// Only track FPS if this image matches the current slider position
if pos == current_slider_pos {
    crate::widgets::slider_image_shader::record_slider_frame();
}
```

This ensures we only count images that are actually displayed, ignoring async loads for positions the slider has already moved past.

### 3. Simplified Performance Tracker

Removed per-render timing and kept only:
- **Prepare time tracking**: Measures atlas upload and GPU resource creation
- **Frame timestamp tracking**: For FPS calculation when images load

## Changes

### `src/widgets/slider_image_shader.rs`
- Fixed `calculate_fps()` to use `len()` instead of `len() - 1`
- Removed `start_render()` and `end_render()` methods
- Added `record_frame()` method called from message handler
- Simplified `SliderPerfTracker` struct (removed render-time fields)
- Updated `get_slider_perf_stats()` to return `(fps, prepare_ms)` instead of `(fps, prepare_ms, render_ms)`

### `src/app/message_handlers.rs`
- Get current slider position before processing loaded image
- Only call `record_slider_frame()` if loaded image position matches current slider position
- This filters out stale async loads that complete after the slider has moved

### `src/ui.rs`
- Updated FPS display to show only FPS and prepare time: `"Slider: X.X FPS (Prep: X.XXms)"`

## Expected Behavior

- **During slider movement**: FPS should reflect actual image loading rate (not render loop rate)
- **Prepare time**: Should show average time spent in `prepare()` (atlas upload + GPU resources)
- **FPS calculation**: Should match the existing Image FPS calculation methodology

## Testing Notes

Test with:
- Small images (should show higher FPS, lower prepare time)
- Large images 1080p+ (will show lower FPS, higher prepare time)
- Rapid slider movement (FPS will drop as image loading can't keep up)

The "Image: 0.0 FPS" issue during slider navigation is expected because the atlas-based rendering bypasses the normal `iced_wgpu` image tracking. The slider-specific FPS display is now the correct metric during slider movement.

