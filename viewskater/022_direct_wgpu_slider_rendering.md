# Direct WGPU Slider Rendering - Design Concept

**Date:** 2025-11-01
**Status:** 💡 Proposed (Not Implemented)

## Current Architecture

Currently, slider navigation uses two different rendering paths:

1. **Normal Mode (ImageShader)**: Direct WGPU rendering via custom shaders
   - Uses `Scene` → `TexturePipeline` → WGPU texture
   - Full control over texture lifecycle
   - Supports zoom/pan interactions

2. **Slider Mode (Iced Viewer Widget)**: Iced's image widget
   - Uses `iced::widget::image::Handle::from_bytes()`
   - Iced manages texture upload via internal atlas
   - Simple but less control over caching

This dual-path approach works but creates inconsistency and limits optimization opportunities.

## Proposed Architecture: Unified WGPU Rendering

### Goal

Eliminate dependency on Iced's Viewer widget for slider rendering. Use direct WGPU rendering for both slider and normal modes, providing:

- Unified rendering path across all navigation modes
- Full control over texture caching strategy
- Better performance through optimized GPU texture management
- Cleaner architecture with less code duplication

### Design Approach

#### 1. Slider Texture Cache

Create a dedicated texture cache for slider images with LRU eviction:

```rust
pub struct SliderTextureCache {
    /// Cached textures indexed by image position
    textures: HashMap<usize, Arc<wgpu::Texture>>,

    /// LRU queue for eviction (most recent at back)
    lru_queue: VecDeque<usize>,

    /// Maximum number of textures to keep resident
    max_cached: usize,  // e.g., 10-20 images

    /// Device and queue for texture operations
    device: Arc<wgpu::Device>,
    queue: Arc<wgpu::Queue>,
}

impl SliderTextureCache {
    /// Get texture for position, creating if needed
    pub fn get_or_create(&mut self, pos: usize, bytes: &[u8]) -> Arc<wgpu::Texture> {
        // Return cached if available
        if let Some(texture) = self.textures.get(&pos) {
            self.update_lru(pos);
            return texture.clone();
        }

        // Create new texture
        let texture = self.create_texture_from_bytes(bytes);

        // Evict oldest if cache full
        if self.textures.len() >= self.max_cached {
            if let Some(oldest) = self.lru_queue.pop_front() {
                self.textures.remove(&oldest);
            }
        }

        // Cache and return
        self.textures.insert(pos, texture.clone());
        self.lru_queue.push_back(pos);
        texture
    }

    fn create_texture_from_bytes(&self, bytes: &[u8]) -> Arc<wgpu::Texture> {
        // Decode image and upload to GPU
        // Similar to Scene::from_bytes() but optimized for slider
        // ...
    }
}
```

#### 2. Lightweight Slider Shader Widget

Create a minimal shader widget for slider rendering (no interaction needed):

```rust
pub struct SliderImageShader<Message> {
    texture: Arc<wgpu::Texture>,
    width: Length,
    height: Length,
    content_fit: ContentFit,
    _phantom: PhantomData<Message>,
}

impl<Message> Widget for SliderImageShader<Message> {
    // Simple passthrough rendering
    // No mouse/zoom interaction handling
    // Just display the texture efficiently
}
```

#### 3. Modified Slider Loading Flow

**Current Flow:**
```rust
// In create_async_image_widget_task()
let bytes = read_image_bytes();
let dimensions = extract_dimensions(&bytes);
let handle = Handle::from_bytes(bytes);  // Iced manages upload
return (pane_idx, pos, handle, dimensions);

// In UI
if slider_mode {
    Viewer::new(slider_image)  // Iced renders
}
```

**Proposed Flow:**
```rust
// In create_async_slider_texture_task()
let bytes = read_image_bytes();
let dimensions = extract_dimensions(&bytes);
// Create Scene directly (or just return bytes + dims)
return (pane_idx, pos, bytes, dimensions);

// In SliderImageLoaded handler
pane.slider_texture_cache.get_or_create(pos, &bytes);

// In UI
if slider_mode {
    SliderImageShader::new(pane.slider_texture_cache.get(pos))
}
```

### Performance Analysis

#### Current Approach
- ✅ Simple implementation
- ⚠️ Iced atlas has overhead
- ⚠️ No control over texture eviction
- ⚠️ Handle → Atlas → Texture indirection

#### Direct WGPU Approach
- ✅ Unified rendering path
- ✅ Custom LRU cache keeps nearby images resident
- ✅ Direct texture rendering (no atlas lookup)
- ✅ Can pre-warm cache (load adjacent images ahead of time)
- ✅ Better control over GPU memory usage
- ⚠️ Slightly more complex implementation

**Performance Gain:** Likely 10-30% faster slider navigation due to:
- Eliminated Handle → Atlas lookup
- Better cache locality (LRU keeps nearby images)
- Opportunity for prefetching adjacent images

### Implementation Phases

#### Phase 1: Core Infrastructure
1. Create `SliderTextureCache` module
2. Implement texture creation from bytes
3. Add LRU eviction logic
4. Add to `Pane` struct

#### Phase 2: Slider Widget
1. Create `SliderImageShader` widget
2. Implement simple WGPU rendering
3. Handle ContentFit::Contain layout
4. Test basic rendering

#### Phase 3: Integration
1. Modify slider loading to populate texture cache
2. Update UI to use `SliderImageShader`
3. Remove dependency on Iced Viewer widget
4. Clean up old Handle-based code

#### Phase 4: Optimization
1. Implement prefetching (load N images ahead)
2. Tune cache size based on memory
3. Add metrics/logging for cache hit rate
4. Consider texture atlas for small images

### Code Locations

**Files to Create:**
- `src/slider_texture_cache.rs` - Texture cache implementation
- `src/widgets/slider_image_shader.rs` - Minimal shader widget

**Files to Modify:**
- `src/pane.rs` - Add `slider_texture_cache` field
- `src/navigation_slider.rs` - Modify loading to populate cache
- `src/ui.rs` - Use `SliderImageShader` instead of `Viewer`
- `src/app.rs` - Handle texture cache updates

### Iced Atlas Integration

The key insight is to study how Iced's atlas works and adapt the useful parts:

**What to Learn from Iced:**
- Texture allocation strategy
- Efficient texture upload (staging buffer usage)
- Texture format selection (RGBA8 vs compressed)

**What to Customize:**
- Cache eviction policy (LRU vs Iced's approach)
- When to upload (async vs sync)
- Memory limits (tune for slider use case)

**Reference Code:**
- `iced_wgpu/src/image/atlas.rs` - Atlas allocation
- `iced_wgpu/src/image/mod.rs` - Texture pipeline

### Benefits Beyond Performance

1. **Consistency**: One rendering path for all modes
2. **Maintainability**: Less code duplication
3. **Flexibility**: Easy to add custom effects (blur, color correction) in shader
4. **Memory Control**: Explicit control over GPU memory usage
5. **Future Features**: Easier to add thumbnail strips, image previews, etc.

### Risks and Mitigations

**Risk**: Texture upload timing
- **Mitigation**: Use async texture creation, similar to current approach

**Risk**: Increased GPU memory usage
- **Mitigation**: Tune cache size, implement eviction policy

**Risk**: Complexity increase
- **Mitigation**: Good abstractions (`SliderTextureCache` encapsulates complexity)

## Conclusion

This approach is worth pursuing as a follow-up PR. It provides:
- Better performance through optimized caching
- Cleaner architecture with unified rendering
- More control for future optimizations

The "copy and paste in a nice way" approach for Iced's atlas logic is totally valid - abstract the useful patterns into a clean `SliderTextureCache` module tailored for slider navigation patterns.

## Next Steps

1. Create separate branch for this work
2. Implement `SliderTextureCache` as standalone module
3. Benchmark current vs proposed approach
4. Iterate on cache size and prefetching strategy
5. Submit as follow-up PR after COCO visualization is merged
