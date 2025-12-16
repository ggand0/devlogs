# Nearest-Neighbor Filter Toggle Implementation

**Date:** 2025-11-19
**Issue:** #64
**Branch:** `feat/filter-toggle`

## Overview

Implemented a nearest-neighbor texture filtering toggle for pixel art viewing. This allows users to switch between smooth (linear) and sharp (nearest-neighbor) scaling modes, which is particularly useful when viewing small pixel art images.

## Background

A user requested (GitHub issue #64) a nearest-neighbor view mode for small images. The default linear filtering causes pixel art to appear blurry when zoomed in, while nearest-neighbor filtering preserves the sharp, blocky appearance that pixel artists intend.

### FilterMode Comparison
- **Linear (default):** Smooth, interpolated pixels when zoomed - good for photographs
- **Nearest:** Sharp, blocky pixels when zoomed - ideal for pixel art

## Implementation

### Files Modified

1. **src/settings.rs** - Added `nearest_neighbor_filter: bool` field with YAML persistence
2. **src/app.rs** - Added field to `DataViewer` struct, initialized from settings
3. **src/app/message.rs** - Added `ToggleNearestNeighborFilter(bool)` message variant
4. **src/app/message_handlers.rs** - Handler that updates state and reloads directories
5. **src/settings_modal.rs** - Toggle UI in General tab
6. **src/ui.rs** - Pass filter setting through UI builder chain
7. **src/pane.rs** - Pass filter to ImageShader and Viewer widgets
8. **src/widgets/shader/image_shader.rs** - Added `use_nearest_filter` field and builder method
9. **src/widgets/shader/texture_pipeline.rs** - Accept filter mode, create appropriate sampler

### Architecture

The filter mode flows through the following chain:
```
YAML settings → UserSettings → DataViewer.nearest_neighbor_filter
    → ui.rs build functions → pane.build_ui_container()
    → ImageShader.use_nearest_filter() → TexturePipeline (wgpu sampler)
```

### Key Technical Details

**TexturePipeline sampler creation:**
```rust
let filter_mode = if use_nearest_filter {
    wgpu::FilterMode::Nearest
} else {
    wgpu::FilterMode::Linear
};

let sampler = device.create_sampler(&wgpu::SamplerDescriptor {
    mag_filter: filter_mode,
    min_filter: filter_mode,
    // ...
});
```

**Pipeline cache keys** now include filter mode to prevent reusing wrong samplers:
```rust
let pipeline_key = format!("img_pipeline_{:.4}_{:.4}_{:.4}_{:.4}_{}",
    bounds_relative.0, bounds_relative.1,
    bounds_relative.2, bounds_relative.3,
    if self.use_nearest_filter { "nearest" } else { "linear" });
```

## Bugs Encountered and Fixed

### Bug 1: Filter didn't apply immediately after toggle
**Problem:** Toggling the setting updated app state but existing GPU pipelines kept the old sampler.

**Solution:** Reload the current directory when toggling to force pipeline recreation:
```rust
Message::ToggleNearestNeighborFilter(enabled) => {
    app.nearest_neighbor_filter = enabled;
    // Force reload to apply filter immediately
    for pane_index in 0..app.panes.len() {
        if let Some(dir_path) = app.panes[pane_index].directory_path.clone() {
            app.initialize_dir_path(&PathBuf::from(dir_path), pane_index);
        }
    }
    Task::none()
}
```

### Bug 2: Pipeline cache reusing wrong filter mode
**Problem:** Pipelines were cached by bounds only, so changing filter mode would reuse the old pipeline with wrong sampler.

**Solution:** Include filter mode in pipeline cache key (see above).

### Bug 3: Filter always false in SinglePane mode (ROOT CAUSE)
**Problem:** The `use_nearest_filter` value was always `false` even when the setting was `true`.

**Investigation:** Added extensive debug logging throughout the chain:
- Settings loading showed `nearest_neighbor_filter=true` ✓
- App initialization showed correct value ✓
- But ImageShader always received `false`

**Root Cause:** The **SinglePane UI path** in `ui.rs` was not calling `.use_nearest_filter()` on the ImageShader widget. The dual-pane paths (`build_ui_dual_pane_slider1/2`) were correctly passing the value, but the single-pane code path in `build_ui()` was missing it entirely.

**Fix:** Added `.use_nearest_filter(app.nearest_neighbor_filter)` to both `#[cfg(feature = "coco")]` and `#[cfg(not(feature = "coco"))]` branches in the SinglePane section of `build_ui()`:

```rust
// Before (broken):
let shader = ImageShader::new(Some(scene))
    .width(Length::Fill)
    .height(Length::Fill)
    .content_fit(iced_winit::core::ContentFit::Contain)
    .horizontal_split(false)
    .with_interaction_state(...)
    .double_click_threshold_ms(...);

// After (fixed):
let shader = ImageShader::new(Some(scene))
    .width(Length::Fill)
    .height(Length::Fill)
    .content_fit(iced_winit::core::ContentFit::Contain)
    .horizontal_split(false)
    .with_interaction_state(...)
    .double_click_threshold_ms(...)
    .use_nearest_filter(app.nearest_neighbor_filter);  // Added
```

## Debugging Approach

The debugging process involved adding logging at each stage:
1. Settings loading (`settings.rs`)
2. App initialization (`app.rs`)
3. Toggle message handler (`message_handlers.rs`)
4. UI builder functions (`ui.rs`)
5. Pane UI container (`pane.rs`)
6. ImageShader primitive creation (`image_shader.rs`)

This revealed that the value was correct until `ui.rs`, where the SinglePane path diverged from dual-pane and didn't pass the setting through.

## Lessons Learned

1. **Multiple code paths:** When adding a new setting that affects rendering, ensure ALL UI code paths pass the value (single pane, dual pane with footer, dual pane without footer, etc.)

2. **Pipeline caching:** GPU pipeline caches must include ALL parameters that affect rendering output in their keys, not just geometric properties.

3. **Systematic debugging:** When a value "isn't working," trace it through the entire chain with logging to find exactly where it gets lost.

## Testing

- Verified filter toggle appears in Settings > General tab
- Verified toggling immediately changes rendering (after directory reload)
- Verified setting persists to YAML and loads on restart
- Tested with 16x16 pixel art PNG - nearest-neighbor shows sharp pixels, linear shows blurry interpolation
