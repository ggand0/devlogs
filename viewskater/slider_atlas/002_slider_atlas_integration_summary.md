# Slider Atlas Integration - Quick Reference

**Date:** 2025-11-03
**Full Plan:** See `001_slider_atlas_integration_plan.md`

---

## TL;DR

**Problem**: Visual jump when releasing slider because rendering switches from iced's Viewer widget to direct wgpu ImageShader.

**Solution**: Bring iced_wgpu's atlas-based rendering directly into viewskater as a self-contained module optimized for slider navigation. Both slider and normal modes will use compatible rendering pipelines with shared shader math for pixel-perfect alignment.

---

## Architecture Overview

### Current (Dual Path - Causes Jump)

```
Slider Mode:  Handle → iced::Viewer → iced's atlas → Screen
Normal Mode:  Scene → ImageShader → Direct texture → Screen
                      ↑ Different rendering path causes jump ↑
```

### Proposed (Unified - No Jump)

```
Slider Mode:  Bytes → SliderAtlasCache → Atlas → AtlasTexturePipeline → Screen
                                                           ↓
Normal Mode:  Scene → ImageShader → Direct texture → TexturePipeline → Screen
                      ↑ Same shader math, pixel-perfect alignment ↑
```

---

## Key Components

### 1. SliderAtlasCache
- **Location**: `src/slider_atlas/cache.rs`
- **Purpose**: Manages atlas uploads with slider-optimized LRU eviction
- **Size**: ~20 images cached (configurable)
- **Memory**: 2MB per layer with BC1 compression

```rust
pub struct SliderAtlasCache {
    atlas: Atlas,
    entries: FxHashMap<ImageKey, CachedEntry>,
    lru: VecDeque<ImageKey>,
    max_entries: usize,
}

impl SliderAtlasCache {
    pub fn get_or_upload(&mut self, pane_idx: usize, image_idx: usize, bytes: &[u8]) 
        -> Result<AtlasImageInfo, SliderAtlasError>;
}
```

### 2. Atlas (from iced_wgpu)
- **Location**: `src/slider_atlas/atlas.rs`
- **Purpose**: 2048x2048 texture array allocation
- **Features**: Dynamic growth, fragmentation support, BC1 compression

### 3. SliderImageShader Widget
- **Location**: `src/widgets/slider_image_shader.rs`
- **Purpose**: Renders images from atlas during slider movement
- **Interface**: Same as current Viewer widget

```rust
pub struct SliderImageShader<Message> {
    pane_idx: usize,
    image_idx: usize,
    // Image bytes passed from Pane
}

// Usage in pane.rs
if is_slider_moving {
    SliderImageShader::new(self.pane_id, self.img_cache.current_index, bytes)
} else {
    ImageShader::new(Some(&self.scene))  // Existing
}
```

### 4. Unified Shader Math
- **Location**: `src/widgets/shader/unified_image_shader.wgsl`
- **Purpose**: Shared vertex/fragment shader for both atlas and direct rendering
- **Result**: Pixel-perfect alignment, no visual jump

---

## Implementation Phases

| Phase | Duration | Output |
|-------|----------|--------|
| 1. Atlas Module | 3-4 days | Atlas allocation working |
| 2. Slider Cache | 2-3 days | LRU cache functional |
| 3. Rendering Pipeline | 2-3 days | Atlas → Screen rendering |
| 4. Widget Integration | 3-4 days | SliderImageShader widget |
| 5. Message Handlers | 2-3 days | End-to-end flow working |
| 6. Alignment Fix | 2-3 days | No visual jump |
| 7. Testing | 3-4 days | Full validation |
| **Total** | **~20-25 days** | **Feature complete** |

---

## Files to Create

```
src/
├── slider_atlas/
│   ├── mod.rs           // Public API
│   ├── atlas.rs         // 2048x2048 texture array (from iced)
│   ├── allocator.rs     // Guillotiere wrapper (from iced)
│   ├── entry.rs         // Atlas entry types (from iced)
│   ├── cache.rs         // LRU cache for slider (NEW)
│   ├── pipeline.rs      // Atlas rendering pipeline (NEW)
│   └── shader.wgsl      // Atlas sampling shader (from iced)
└── widgets/
    └── slider_image_shader.rs  // Widget using atlas (NEW)
```

## Files to Modify

```
src/
├── app/
│   └── message_handlers.rs  // Add SliderImageAtlasLoaded handler
├── navigation_slider.rs      // Add atlas-based loading task
├── pane.rs                   // Add slider_image_bytes field
└── ui.rs                     // Use SliderImageShader widget
```

---

## Critical Design Decisions

### 1. Image Bytes Access
**Problem**: Widget needs bytes at render time, but we're in rendering context without access to app state.

**Solution**: Pre-load bytes in async task, store in `Pane::slider_image_bytes`, pass to widget.

```rust
// In navigation_slider.rs
pub async fn create_async_slider_atlas_task(...) -> Result<(usize, usize, Vec<u8>, (u32, u32)), ...> {
    let bytes = load_bytes(...);
    let dimensions = extract_dimensions(&bytes);
    Ok((pane_idx, pos, bytes, dimensions))  // Return bytes, not Handle
}

// In pane.rs
pub struct Pane {
    pub slider_image_bytes: Option<Vec<u8>>,  // NEW
    pub slider_image_dimensions: Option<(u32, u32)>,
}

// In message_handlers.rs
Message::SliderImageAtlasLoaded(Ok((pane_idx, pos, bytes, dims))) => {
    app.panes[pane_idx].slider_image_bytes = Some(bytes);
    app.panes[pane_idx].slider_image_dimensions = Some(dims);
}
```

### 2. Rendering Alignment
**Problem**: Subtle differences in shader math cause visual jump.

**Solution**: Share exact shader code and uniform calculations between both paths.

```rust
// Both TexturePipeline and AtlasTexturePipeline use:
struct ImageUniforms {
    content_bounds: vec4<f32>,  // EXACT same calculation
    viewport_size: vec2<f32>,
}

// Shader: unified_image_shader.wgsl
// Same vertex calculation for both direct and atlas textures
```

### 3. LRU Eviction Strategy
**Problem**: How many images to cache? When to evict?

**Solution**: 
- Cache size: 20 images (tunable)
- Eviction: Strict LRU (oldest access time)
- Optional: Predictive prefetching (load adjacent images)

---

## Performance Targets

| Metric | Target | Current |
|--------|--------|---------|
| Slider FPS | ≥60 fps | ~60 fps |
| Atlas Upload | <5ms | N/A |
| Cache Hit Rate | >90% | N/A |
| Memory (Atlas) | <100MB | N/A |
| Visual Jump | 0px | ~1-2px |

---

## Risk Mitigation

### Memory Usage
- **Risk**: Atlas consumes significant VRAM
- **Mitigation**: Use BC1 compression (8:1), monitor usage, aggressive eviction

### BC1 Quality
- **Risk**: Compression artifacts on some images
- **Mitigation**: Make configurable, add quality slider, test on diverse images

### Integration Complexity
- **Risk**: Widget needs bytes at render time
- **Mitigation**: Pre-load in async task, document lifetime clearly

---

## Testing Strategy

### Unit Tests
- Atlas allocation and growth
- LRU eviction correctness
- Cache hit/miss behavior

### Visual Tests
- Grid pattern alignment test (no jump)
- COCO annotation alignment (pixel-perfect)
- Large image fragmentation (>2048px)
- BC1 compression quality (photos, graphics, text)

### Performance Tests
- Upload benchmark (<5ms)
- Slider navigation (60fps)
- Cache hit rate (>90%)
- Memory usage monitoring

---

## Success Criteria

✅ **No visual jump** when releasing slider  
✅ **Pixel-perfect alignment** with COCO annotations  
✅ **Performance parity** with existing Viewer widget  
✅ **Memory usage** <100MB for atlas  
✅ **Cache hit rate** >90% during typical usage  
✅ **All tests passing** (unit, visual, performance)  

---

## Next Steps

1. ✅ Review and approve plan
2. 🔄 Create feature branch: `feature/slider-atlas-integration`
3. ⏳ Phase 1: Port atlas allocation code from iced
4. ⏳ Phase 2: Implement slider-optimized LRU cache
5. ⏳ Phase 3: Create atlas rendering pipeline
6. ⏳ Phase 4: Build SliderImageShader widget
7. ⏳ Phase 5: Integrate with message handlers
8. ⏳ Phase 6: Fix rendering alignment
9. ⏳ Phase 7: Comprehensive testing

---

## References

- **Full Plan**: `110325_slider_atlas_integration_plan.md`
- **Iced Atlas Code**: `/home/gota/ggando/rust_gui/iced/wgpu/src/image/atlas.rs`
- **Current Viewer**: `src/widgets/viewer.rs`
- **Current ImageShader**: `src/widgets/shader/image_shader.rs`
- **Slider Navigation**: `src/navigation_slider.rs`

---

**Questions? Start with Phase 1!** 🚀

