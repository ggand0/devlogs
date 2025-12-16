# Slider Rendering Atlas Integration Plan

**Date:** 2025-11-03
**Status:** 📋 Planning Phase  
**Goal**: Integrate iced_wgpu's atlas-based rendering directly into viewskater for slider navigation, eliminating the visual jump between slider and normal rendering modes.

---

## Table of Contents

1. [Current Architecture Analysis](#current-architecture-analysis)
2. [Problem Statement](#problem-statement)
3. [Proposed Solution](#proposed-solution)
4. [Implementation Strategy](#implementation-strategy)
5. [Detailed Phase Breakdown](#detailed-phase-breakdown)
6. [Risk Analysis & Mitigations](#risk-analysis--mitigations)
7. [Testing Strategy](#testing-strategy)
8. [Performance Considerations](#performance-considerations)
9. [Future Enhancements](#future-enhancements)

---

## Current Architecture Analysis

### Dual Rendering Path

ViewSkater currently uses two separate rendering paths:

#### 1. Slider Mode (Iced Viewer Widget)
```rust
// In navigation_slider.rs
pub async fn create_async_image_widget_task(...) -> Result<(usize, usize, Handle, (u32, u32)), ...> {
    let bytes = read_image_bytes(...);
    let handle = iced::widget::image::Handle::from_bytes(bytes); // ← Iced manages upload
    Ok((pane_idx, pos, handle, dimensions))
}

// In pane.rs:673-686
if is_slider_moving && self.slider_image.is_some() {
    center(viewer::Viewer::new(image_handle)
        .content_fit(iced_winit::core::ContentFit::Contain))
}
```

**How it works:**
- `Handle::from_bytes()` creates an iced image handle
- Iced's `raster::Cache` loads the image as `Memory::Host` 
- On first render, `upload()` is called → uploads to `Atlas` via `atlas.upload()`
- Returns `atlas::Entry` (location in 2048x2048 texture atlas)
- Subsequent renders use cached atlas entry (very fast)

#### 2. Normal Mode (Direct WGPU via ImageShader)
```rust
// In pane.rs:687-722
else if let Some(scene) = &self.scene {
    ImageShader::new(Some(scene))
        .width(Length::Fill)
        .height(Length::Fill)
        .content_fit(iced_winit::core::ContentFit::Contain)
        .with_interaction_state(self.mouse_wheel_zoom, self.ctrl_pressed)
}
```

**How it works:**
- Uses custom `Scene` enum (TextureScene or CpuScene)
- Scene holds direct `Arc<wgpu::Texture>` reference
- Custom `TexturePipeline` renders texture to screen
- Full control over zoom, pan, texture lifecycle
- No atlas indirection

### The Visual Jump Problem

**When slider is released** (`Message::SliderReleased`):
1. `app.is_slider_moving` set to `false` (message_handlers.rs:418)
2. UI switches from `Viewer` widget to `ImageShader` widget (ui.rs:287-311)
3. Slight rendering differences cause visible "jump":
   - Different texture sampling/filtering
   - Potential sub-pixel alignment differences
   - Different ContentFit calculation paths
4. **Especially noticeable with COCO annotations** overlaid on top, as they stay in place while image shifts

### Iced_wgpu Atlas System Deep Dive

**Key Components** (from `/home/gota/ggando/rust_gui/iced/wgpu/src/image/`):

1. **Atlas** (`atlas.rs:24-108`)
   - 2048x2048 texture array with multiple layers
   - Format: `Rgba8UnormSrgb` or `Bc1RgbaUnormSrgb` based on `CompressionStrategy`
   - Efficient allocation via `guillotiere` allocator
   - Grows automatically by adding layers when full
   - Supports fragmented allocation for large images (>2048px)

2. **Cache** (`cache.rs:7-88`)
   - Main entry point: `upload_raster()` 
   - Delegates to `raster::Cache` for image loading
   - Manages atlas lifecycle
   - `trim()` removes unused entries based on LRU

3. **Raster Cache** (`raster.rs:38-132`)
   - `FxHashMap<image::Id, Memory>` for O(1) lookup
   - `Memory` enum: `Host` (CPU) or `Device` (GPU atlas entry)
   - First load: `Memory::Host(ImageBuffer)`
   - First render: uploads to atlas → `Memory::Device(atlas::Entry)`
   - LRU tracking via `hits: FxHashSet<image::Id>`

4. **Atlas Entry** (`atlas/entry.rs`)
   - `Entry::Contiguous` for single allocation
   - `Entry::Fragmented` for images split across multiple atlas slots
   - Stores position (x, y, layer) and size

**Performance Characteristics:**
- Initial upload: ~2-5ms for typical images (depends on size)
- Cached render: <0.1ms (just bind group + draw call)
- Memory overhead: 4 bytes per pixel (RGBA8) or 0.5 bytes per pixel (BC1)
- Atlas grows dynamically (no hard limit on image count)

---

## Problem Statement

### Core Issues

1. **Visual Discontinuity**: Slider release causes visible jump due to rendering path switch
2. **Maintenance Burden**: Custom BC1 support required forking iced (harder to update)
3. **Code Duplication**: Two separate rendering pipelines for the same images
4. **COCO Alignment**: Annotations stay fixed while image shifts, breaking visual coherence

### Why This Matters

- **User Experience**: Jump is jarring, especially during COCO annotation workflows
- **Technical Debt**: Maintaining two rendering paths increases complexity
- **Future-Proofing**: Unified pipeline easier to extend (e.g., thumbnail strips, previews)

---

## Proposed Solution

### High-Level Approach

**Bring iced_wgpu's atlas rendering directly into viewskater as a self-contained module**, adapted specifically for slider navigation patterns.

**Key Insight**: We don't need to replace iced's `Viewer` widget. Instead, we create a parallel "slider-optimized" atlas system that:
1. Uses the same atlas upload logic (proven fast)
2. Renders with the same shader as ImageShader (consistent appearance)
3. Tailored LRU policy for slider access patterns
4. Lives entirely in viewskater codebase

This gives us:
- ✅ Consistency: Same rendering code for slider + normal modes
- ✅ Performance: Atlas caching proven by iced (fast slider navigation)
- ✅ Control: Can optimize for slider-specific patterns
- ✅ Maintainability: Self-contained in viewskater, no iced fork dependency for this feature

---

## Implementation Strategy

### Module Structure

```
src/
├── slider_atlas/
│   ├── mod.rs           // Public API
│   ├── atlas.rs         // Atlas allocation (adapted from iced)
│   ├── allocator.rs     // Guillotiere wrapper (from iced)
│   ├── entry.rs         // Atlas entry types (from iced)
│   ├── cache.rs         // Slider-optimized cache with LRU
│   ├── pipeline.rs      // Rendering pipeline (reuse TexturePipeline)
│   └── shader.wgsl      // Atlas texture sampling shader
├── widgets/
│   └── slider_image_shader.rs  // Widget using SliderAtlasCache
```

### Core Types

```rust
// slider_atlas/mod.rs
pub struct SliderAtlasCache {
    atlas: Atlas,
    entries: FxHashMap<ImageKey, CachedEntry>,
    lru: VecDeque<ImageKey>,  // Most recent at back
    max_entries: usize,        // e.g., 20 images
    device: Arc<wgpu::Device>,
    queue: Arc<wgpu::Queue>,
}

#[derive(Hash, Eq, PartialEq)]
struct ImageKey {
    pane_idx: usize,
    image_idx: usize,
}

struct CachedEntry {
    entry: atlas::Entry,       // Position in atlas
    dimensions: (u32, u32),
    last_access: Instant,
}

impl SliderAtlasCache {
    /// Get or upload image to atlas
    pub fn get_or_upload(&mut self, 
        pane_idx: usize, 
        image_idx: usize, 
        bytes: &[u8]
    ) -> Result<&atlas::Entry, Error> {
        let key = ImageKey { pane_idx, image_idx };
        
        // Check cache
        if let Some(cached) = self.entries.get_mut(&key) {
            cached.last_access = Instant::now();
            self.update_lru(&key);
            return Ok(&cached.entry);
        }
        
        // Decode image
        let img = image::load_from_memory(bytes)?;
        let rgba = img.to_rgba8();
        let (width, height) = rgba.dimensions();
        
        // Upload to atlas
        let mut encoder = self.device.create_command_encoder(&Default::default());
        let entry = self.atlas.upload(&self.device, &mut encoder, width, height, &rgba)?;
        self.queue.submit(Some(encoder.finish()));
        
        // Evict if needed
        if self.entries.len() >= self.max_entries {
            if let Some(oldest_key) = self.lru.pop_front() {
                if let Some(old_entry) = self.entries.remove(&oldest_key) {
                    self.atlas.remove(&old_entry.entry);
                }
            }
        }
        
        // Cache new entry
        self.entries.insert(key, CachedEntry {
            entry,
            dimensions: (width, height),
            last_access: Instant::now(),
        });
        self.lru.push_back(key);
        
        Ok(&self.entries[&key].entry)
    }
    
    fn update_lru(&mut self, key: &ImageKey) {
        // Move to back (most recent)
        if let Some(pos) = self.lru.iter().position(|k| k == key) {
            self.lru.remove(pos);
        }
        self.lru.push_back(*key);
    }
}
```

### Rendering Integration

**Key Decision: Reuse TexturePipeline with atlas as source**

Current ImageShader uses:
```rust
Scene → TexturePipeline → Direct wgpu::Texture → Screen
```

New slider path:
```rust
SliderAtlasCache → atlas::Entry → TexturePipeline → Atlas Texture → Screen
```

**Modified ImageShader** (`widgets/slider_image_shader.rs`):
```rust
pub struct SliderImageShader<Message> {
    pane_idx: usize,
    image_idx: usize,
    width: Length,
    height: Length,
    content_fit: ContentFit,
    _phantom: PhantomData<Message>,
}

impl<Message> Widget for SliderImageShader<Message> {
    fn draw(&self, ...) -> Primitive {
        SliderImagePrimitive {
            pane_idx: self.pane_idx,
            image_idx: self.image_idx,
            bounds,
            content_fit: self.content_fit,
        }
    }
}

impl shader::Primitive for SliderImagePrimitive {
    fn prepare(&self, device, queue, storage, ...) {
        // Get or create SliderAtlasCache in storage
        if !storage.has::<SliderAtlasCache>() {
            storage.store(SliderAtlasCache::new(device, queue));
        }
        let cache = storage.get_mut::<SliderAtlasCache>().unwrap();
        
        // Load image bytes (from pane.img_cache or async load)
        let bytes = load_image_bytes(self.pane_idx, self.image_idx)?;
        
        // Upload to atlas
        let entry = cache.get_or_upload(self.pane_idx, self.image_idx, &bytes)?;
        
        // Store for rendering
        storage.store(PreparedSliderImage {
            entry: entry.clone(),
            dimensions: cache.entries[&ImageKey { 
                pane_idx: self.pane_idx, 
                image_idx: self.image_idx 
            }].dimensions,
        });
        
        // Ensure pipeline exists
        if !storage.has::<AtlasTexturePipeline>() {
            storage.store(AtlasTexturePipeline::new(device, format, &cache.atlas));
        }
    }
    
    fn render(&self, encoder, storage, target, clip_bounds) {
        let pipeline = storage.get::<AtlasTexturePipeline>().unwrap();
        let prepared = storage.get::<PreparedSliderImage>().unwrap();
        
        // Render from atlas
        pipeline.render_atlas_entry(encoder, &prepared.entry, target, clip_bounds);
    }
}
```

### Widget Usage in UI

```rust
// In pane.rs::build_ui_container()
pub fn build_ui_container(&self, is_slider_moving: bool, ...) -> Container<...> {
    if self.dir_loaded {
        if is_slider_moving {
            // NEW: Use SliderImageShader instead of Viewer
            container(center(
                SliderImageShader::new(self.pane_id, self.img_cache.current_index)
                    .width(Length::Fill)
                    .height(Length::Fill)
                    .content_fit(iced_winit::core::ContentFit::Contain)
            ))
        } else {
            // Keep existing ImageShader path
            container(center(
                ImageShader::new(Some(&self.scene))
                    // ... existing setup
            ))
        }
    }
}
```

**Critical Insight**: Both paths now use compatible rendering:
- SliderImageShader → Atlas → AtlasTexturePipeline → Same shader math as TexturePipeline
- ImageShader → Scene → TexturePipeline → Direct texture

The shader calculations can be unified to ensure pixel-perfect alignment.

---

## Detailed Phase Breakdown

### Phase 1: Atlas Module (Core Infrastructure)
**Goal**: Port iced's atlas allocation logic into viewskater

**Files to Create:**
- `src/slider_atlas/mod.rs`
- `src/slider_atlas/atlas.rs` 
- `src/slider_atlas/allocator.rs` (guillotiere wrapper)
- `src/slider_atlas/entry.rs`
- `src/slider_atlas/shader.wgsl`

**Implementation Steps:**

1. **Copy atlas code** from iced_wgpu (`iced/wgpu/src/image/atlas.rs`):
   ```rust
   // slider_atlas/atlas.rs
   pub struct Atlas {
       texture: wgpu::Texture,
       texture_view: wgpu::TextureView,
       texture_bind_group: wgpu::BindGroup,
       texture_layout: Arc<wgpu::BindGroupLayout>,
       layers: Vec<Layer>,
       compression_strategy: CompressionStrategy,
   }
   
   impl Atlas {
       pub fn new(device: &wgpu::Device, backend: wgpu::Backend, ...) -> Self {
           // Create 2048x2048 texture array
           // Setup bind group layout
           // Initialize first layer
       }
       
       pub fn upload(&mut self, device, encoder, width, height, data) -> Option<Entry> {
           // Allocate space in atlas
           // Handle padding for COPY_BYTES_PER_ROW_ALIGNMENT
           // Upload data to texture
           // Return Entry with position
       }
       
       pub fn remove(&mut self, entry: &Entry) {
           // Deallocate space
       }
   }
   ```

2. **Adapt allocation logic**:
   - Keep guillotiere allocator (proven efficient)
   - Simplify: Remove SVG support (only need raster images)
   - Keep fragmentation support (images >2048px)
   - Keep layer growth logic

3. **Add compression support**:
   ```rust
   match self.compression_strategy {
       CompressionStrategy::None => {
           // RGBA8 upload (existing iced code)
       }
       CompressionStrategy::Bc1 => {
           // Use texpresso for BC1 compression
           // Upload compressed data
       }
   }
   ```

**Testing:**
- Unit test: Upload single image → verify entry returned
- Unit test: Upload until atlas full → verify new layer created
- Unit test: Upload 3000x3000 image → verify fragmentation works
- Visual test: Render atlas contents to verify uploads correct

**Success Criteria:**
- Atlas can allocate and upload images
- Handles images of all sizes (<2048, =2048, >2048)
- BC1 compression works
- Layer growth functions correctly

---

### Phase 2: Slider Cache (LRU Management)
**Goal**: Create slider-optimized cache with efficient LRU eviction

**File to Create:**
- `src/slider_atlas/cache.rs`

**Implementation:**

```rust
pub struct SliderAtlasCache {
    atlas: Atlas,
    entries: FxHashMap<ImageKey, CachedEntry>,
    lru: VecDeque<ImageKey>,
    max_entries: usize,
    device: Arc<wgpu::Device>,
    queue: Arc<wgpu::Queue>,
}

#[derive(Hash, Eq, PartialEq, Copy, Clone)]
struct ImageKey {
    pane_idx: usize,
    image_idx: usize,
}

struct CachedEntry {
    entry: atlas::Entry,
    dimensions: (u32, u32),
    last_access: Instant,
}

impl SliderAtlasCache {
    pub fn new(device: Arc<wgpu::Device>, queue: Arc<wgpu::Queue>, max_entries: usize) -> Self {
        let bind_group_layout = device.create_bind_group_layout(&wgpu::BindGroupLayoutDescriptor {
            label: Some("slider_atlas_layout"),
            entries: &[wgpu::BindGroupLayoutEntry {
                binding: 0,
                visibility: wgpu::ShaderStages::FRAGMENT,
                ty: wgpu::BindingType::Texture {
                    sample_type: wgpu::TextureSampleType::Float { filterable: true },
                    view_dimension: wgpu::TextureViewDimension::D2Array,
                    multisampled: false,
                },
                count: None,
            }],
        });
        
        let atlas = Atlas::new(
            &device,
            wgpu::Backend::Vulkan, // Will be set properly
            Arc::new(bind_group_layout),
            CompressionStrategy::Bc1,
        );
        
        Self {
            atlas,
            entries: FxHashMap::default(),
            lru: VecDeque::with_capacity(max_entries),
            max_entries,
            device,
            queue,
        }
    }
    
    pub fn get_or_upload(
        &mut self,
        pane_idx: usize,
        image_idx: usize,
        bytes: &[u8],
    ) -> Result<AtlasImageInfo, SliderAtlasError> {
        let key = ImageKey { pane_idx, image_idx };
        
        // Cache hit
        if let Some(cached) = self.entries.get_mut(&key) {
            cached.last_access = Instant::now();
            self.update_lru(&key);
            return Ok(AtlasImageInfo {
                entry: cached.entry.clone(),
                dimensions: cached.dimensions,
            });
        }
        
        // Cache miss - load and upload
        let img = image::load_from_memory(bytes)
            .map_err(|e| SliderAtlasError::ImageDecode(e))?;
        let rgba = img.to_rgba8();
        let (width, height) = rgba.dimensions();
        
        // Upload to atlas
        let mut encoder = self.device.create_command_encoder(&wgpu::CommandEncoderDescriptor {
            label: Some("slider_atlas_upload"),
        });
        
        let entry = self.atlas.upload(&self.device, &mut encoder, width, height, rgba.as_raw())
            .ok_or(SliderAtlasError::AtlasFull)?;
        
        self.queue.submit(Some(encoder.finish()));
        
        // LRU eviction if needed
        self.evict_if_needed();
        
        // Cache new entry
        let cached_entry = CachedEntry {
            entry: entry.clone(),
            dimensions: (width, height),
            last_access: Instant::now(),
        };
        self.entries.insert(key, cached_entry);
        self.lru.push_back(key);
        
        Ok(AtlasImageInfo {
            entry,
            dimensions: (width, height),
        })
    }
    
    fn evict_if_needed(&mut self) {
        while self.entries.len() >= self.max_entries {
            if let Some(oldest_key) = self.lru.pop_front() {
                if let Some(old_entry) = self.entries.remove(&oldest_key) {
                    self.atlas.remove(&old_entry.entry);
                    debug!("Evicted atlas entry: pane={}, image={}", 
                           oldest_key.pane_idx, oldest_key.image_idx);
                }
            } else {
                break;
            }
        }
    }
    
    fn update_lru(&mut self, key: &ImageKey) {
        // Remove from current position
        self.lru.retain(|k| k != key);
        // Add to back (most recent)
        self.lru.push_back(*key);
    }
    
    pub fn clear(&mut self) {
        for entry in self.entries.values() {
            self.atlas.remove(&entry.entry);
        }
        self.entries.clear();
        self.lru.clear();
    }
    
    pub fn cache_stats(&self) -> CacheStats {
        CacheStats {
            entries: self.entries.len(),
            max_entries: self.max_entries,
            atlas_layers: self.atlas.layer_count(),
        }
    }
}

pub struct AtlasImageInfo {
    pub entry: atlas::Entry,
    pub dimensions: (u32, u32),
}

#[derive(Debug)]
pub enum SliderAtlasError {
    ImageDecode(image::ImageError),
    AtlasFull,
}
```

**Optimization: Predictive Preloading**

```rust
impl SliderAtlasCache {
    /// Preload adjacent images for smoother slider experience
    pub fn preload_adjacent(
        &mut self,
        pane_idx: usize,
        current_idx: usize,
        image_paths: &[PathSource],
        prefetch_count: usize,
    ) {
        // Load next N images
        for offset in 1..=prefetch_count {
            let idx = current_idx + offset;
            if idx < image_paths.len() {
                if let Ok(bytes) = load_image_bytes(&image_paths[idx]) {
                    let _ = self.get_or_upload(pane_idx, idx, &bytes);
                }
            }
        }
        
        // Load previous N images
        for offset in 1..=prefetch_count {
            if let Some(idx) = current_idx.checked_sub(offset) {
                if let Ok(bytes) = load_image_bytes(&image_paths[idx]) {
                    let _ = self.get_or_upload(pane_idx, idx, &bytes);
                }
            }
        }
    }
}
```

**Testing:**
- Test LRU eviction: Fill cache to max → upload one more → verify oldest removed
- Test cache hit: Upload image → retrieve same image → verify no re-upload
- Test multi-pane: Upload same image_idx for different panes → verify separate entries
- Performance test: Measure cache hit rate during slider movement

**Success Criteria:**
- LRU eviction works correctly
- Cache hit rate >90% during typical slider usage
- No memory leaks (atlas entries properly cleaned up)

---

### Phase 3: Atlas Rendering Pipeline
**Goal**: Create rendering pipeline that uses atlas as texture source

**File to Create:**
- `src/slider_atlas/pipeline.rs`

**Implementation:**

```rust
pub struct AtlasTexturePipeline {
    render_pipeline: wgpu::RenderPipeline,
    bind_group_layout: Arc<wgpu::BindGroupLayout>,
    sampler: wgpu::Sampler,
}

impl AtlasTexturePipeline {
    pub fn new(device: &wgpu::Device, format: wgpu::TextureFormat, atlas: &Atlas) -> Self {
        let bind_group_layout = /* same as TexturePipeline */;
        
        let shader = device.create_shader_module(wgpu::ShaderModuleDescriptor {
            label: Some("atlas_texture_shader"),
            source: wgpu::ShaderSource::Wgsl(include_str!("shader.wgsl").into()),
        });
        
        let pipeline_layout = device.create_pipeline_layout(&wgpu::PipelineLayoutDescriptor {
            label: Some("atlas_texture_pipeline_layout"),
            bind_group_layouts: &[&bind_group_layout],
            push_constant_ranges: &[],
        });
        
        let render_pipeline = device.create_render_pipeline(&wgpu::RenderPipelineDescriptor {
            label: Some("atlas_texture_pipeline"),
            layout: Some(&pipeline_layout),
            vertex: wgpu::VertexState {
                module: &shader,
                entry_point: "vs_main",
                buffers: &[],
            },
            fragment: Some(wgpu::FragmentState {
                module: &shader,
                entry_point: "fs_main",
                targets: &[Some(wgpu::ColorTargetState {
                    format,
                    blend: Some(wgpu::BlendState::REPLACE),
                    write_mask: wgpu::ColorWrites::ALL,
                })],
            }),
            primitive: wgpu::PrimitiveState::default(),
            depth_stencil: None,
            multisample: wgpu::MultisampleState::default(),
            multiview: None,
        });
        
        let sampler = device.create_sampler(&wgpu::SamplerDescriptor {
            label: Some("atlas_sampler"),
            address_mode_u: wgpu::AddressMode::ClampToEdge,
            address_mode_v: wgpu::AddressMode::ClampToEdge,
            mag_filter: wgpu::FilterMode::Linear,
            min_filter: wgpu::FilterMode::Linear,
            ..Default::default()
        });
        
        Self {
            render_pipeline,
            bind_group_layout,
            sampler,
        }
    }
    
    pub fn render(
        &self,
        encoder: &mut wgpu::CommandEncoder,
        atlas_bind_group: &wgpu::BindGroup,
        entry: &atlas::Entry,
        target: &wgpu::TextureView,
        bounds: Rectangle,
        viewport: &Viewport,
    ) {
        let mut render_pass = encoder.begin_render_pass(&wgpu::RenderPassDescriptor {
            label: Some("atlas_texture_render_pass"),
            color_attachments: &[Some(wgpu::RenderPassColorAttachment {
                view: target,
                resolve_target: None,
                ops: wgpu::Operations {
                    load: wgpu::LoadOp::Load, // Don't clear - we're overlaying
                    store: wgpu::StoreOp::Store,
                },
            })],
            depth_stencil_attachment: None,
            timestamp_writes: None,
            occlusion_query_set: None,
        });
        
        render_pass.set_pipeline(&self.render_pipeline);
        render_pass.set_bind_group(0, atlas_bind_group, &[]);
        
        // Set scissor for bounds
        let scale = viewport.scale_factor() as f32;
        render_pass.set_scissor_rect(
            (bounds.x * scale) as u32,
            (bounds.y * scale) as u32,
            (bounds.width * scale) as u32,
            (bounds.height * scale) as u32,
        );
        
        // Draw full-screen quad - shader will sample from atlas entry
        render_pass.draw(0..6, 0..1);
    }
}
```

**Shader** (`slider_atlas/shader.wgsl`):
```wgsl
struct VertexOutput {
    @builtin(position) position: vec4<f32>,
    @location(0) tex_coords: vec2<f32>,
};

@vertex
fn vs_main(@builtin(vertex_index) vertex_index: u32) -> VertexOutput {
    var output: VertexOutput;
    
    // Full-screen triangle
    let x = f32((vertex_index & 1u) << 2u) - 1.0;
    let y = f32((vertex_index & 2u) << 1u) - 1.0;
    
    output.position = vec4<f32>(x, y, 0.0, 1.0);
    output.tex_coords = vec2<f32>((x + 1.0) * 0.5, (1.0 - y) * 0.5);
    
    return output;
}

@group(0) @binding(0) var atlas_texture: texture_2d_array<f32>;
@group(0) @binding(1) var atlas_sampler: sampler;

@fragment
fn fs_main(input: VertexOutput) -> @location(0) vec4<f32> {
    // TODO: Need to pass atlas entry info via push constants or uniforms
    // For now, sample layer 0
    return textureSample(atlas_texture, atlas_sampler, input.tex_coords, 0);
}
```

**Critical TODO**: Pass atlas entry information to shader
- Option 1: Push constants (entry layer, position, size)
- Option 2: Uniform buffer
- Option 3: Separate pipeline per entry (wasteful)

**Recommended: Push Constants**
```rust
// Add to pipeline.rs
pub fn render_with_entry_info(...) {
    // Calculate normalized atlas coordinates from entry
    let (layer, x, y, width, height) = entry.atlas_coords();
    let push_constants = AtlasEntryInfo {
        layer,
        atlas_rect: [
            x as f32 / ATLAS_SIZE as f32,
            y as f32 / ATLAS_SIZE as f32,
            width as f32 / ATLAS_SIZE as f32,
            height as f32 / ATLAS_SIZE as f32,
        ],
    };
    
    render_pass.set_push_constants(
        wgpu::ShaderStages::FRAGMENT,
        0,
        bytemuck::bytes_of(&push_constants),
    );
}
```

**Testing:**
- Visual test: Render single image from atlas → verify correct appearance
- Visual test: Render multiple images in grid → verify no crosstalk
- Visual test: Compare atlas render vs direct texture render → should be identical
- Performance test: Measure render time (should be <0.1ms)

**Success Criteria:**
- Images render correctly from atlas
- No visual artifacts or sampling errors
- Performance matches or exceeds direct texture rendering

---

### Phase 4: Widget Integration
**Goal**: Create `SliderImageShader` widget that uses the atlas cache

**File to Create:**
- `src/widgets/slider_image_shader.rs`

**Implementation:**

```rust
use iced_core::*;
use iced_wgpu::primitive;
use crate::slider_atlas::{SliderAtlasCache, AtlasImageInfo};

pub struct SliderImageShader<Message> {
    pane_idx: usize,
    image_idx: usize,
    width: Length,
    height: Length,
    content_fit: ContentFit,
    _phantom: PhantomData<Message>,
}

impl<Message> SliderImageShader<Message> {
    pub fn new(pane_idx: usize, image_idx: usize) -> Self {
        Self {
            pane_idx,
            image_idx,
            width: Length::Fill,
            height: Length::Fill,
            content_fit: ContentFit::Contain,
            _phantom: PhantomData,
        }
    }
    
    pub fn width(mut self, width: impl Into<Length>) -> Self {
        self.width = width.into();
        self
    }
    
    pub fn height(mut self, height: impl Into<Length>) -> Self {
        self.height = height.into();
        self
    }
    
    pub fn content_fit(mut self, content_fit: ContentFit) -> Self {
        self.content_fit = content_fit;
        self
    }
}

impl<Message, Theme, R> Widget<Message, Theme, R> for SliderImageShader<Message>
where
    R: primitive::Renderer,
{
    fn size(&self) -> Size<Length> {
        Size::new(self.width, self.height)
    }
    
    fn layout(&self, _tree: &mut Tree, _renderer: &R, limits: &layout::Limits) -> layout::Node {
        layout::atomic(limits, self.width, self.height)
    }
    
    fn draw(&self, _tree: &Tree, renderer: &mut R, _theme: &Theme, _style: &renderer::Style, 
            layout: Layout<'_>, _cursor: mouse::Cursor, _viewport: &Rectangle) {
        let bounds = layout.bounds();
        
        let primitive = SliderImagePrimitive {
            pane_idx: self.pane_idx,
            image_idx: self.image_idx,
            bounds,
            content_fit: self.content_fit,
        };
        
        renderer.draw_primitive(bounds, primitive);
    }
}

#[derive(Debug)]
struct SliderImagePrimitive {
    pane_idx: usize,
    image_idx: usize,
    bounds: Rectangle,
    content_fit: ContentFit,
}

impl shader::Primitive for SliderImagePrimitive {
    fn prepare(
        &self,
        device: &wgpu::Device,
        queue: &wgpu::Queue,
        format: wgpu::TextureFormat,
        storage: &mut shader::Storage,
        _bounds: &Rectangle,
        viewport: &Viewport,
    ) {
        // Initialize cache if needed
        if !storage.has::<SliderAtlasCache>() {
            let device = Arc::new(device.clone()); // TODO: Get proper Arc from app
            let queue = Arc::new(queue.clone());
            storage.store(SliderAtlasCache::new(device, queue, 20));
        }
        
        // Get image bytes
        // TODO: Need access to pane.img_cache or async load here
        // This is a key integration point - need to bridge with existing loading system
        let bytes = self.load_image_bytes()?;
        
        // Upload to atlas
        let cache = storage.get_mut::<SliderAtlasCache>().unwrap();
        let info = cache.get_or_upload(self.pane_idx, self.image_idx, &bytes)
            .expect("Failed to upload to atlas");
        
        // Calculate content bounds (same logic as ImageShader)
        let content_bounds = self.calculate_content_bounds(&info, viewport);
        
        // Store prepared info
        storage.store(PreparedSliderImage {
            info,
            content_bounds,
            viewport: viewport.clone(),
        });
        
        // Initialize pipeline if needed
        if !storage.has::<AtlasTexturePipeline>() {
            storage.store(AtlasTexturePipeline::new(device, format));
        }
    }
    
    fn render(
        &self,
        encoder: &mut wgpu::CommandEncoder,
        storage: &shader::Storage,
        target: &wgpu::TextureView,
        clip_bounds: &Rectangle<u32>,
    ) {
        let pipeline = storage.get::<AtlasTexturePipeline>().unwrap();
        let prepared = storage.get::<PreparedSliderImage>().unwrap();
        let cache = storage.get::<SliderAtlasCache>().unwrap();
        
        pipeline.render(
            encoder,
            cache.atlas.bind_group(),
            &prepared.info.entry,
            target,
            prepared.content_bounds,
            &prepared.viewport,
        );
    }
}

impl SliderImagePrimitive {
    fn calculate_content_bounds(&self, info: &AtlasImageInfo, viewport: &Viewport) -> Rectangle {
        let image_size = Size::new(info.dimensions.0 as f32, info.dimensions.1 as f32);
        let bounds_size = self.bounds.size();
        
        // Same ContentFit logic as ImageShader
        let (width, height) = match self.content_fit {
            ContentFit::Contain => {
                let ratio = (bounds_size.width / image_size.width)
                    .min(bounds_size.height / image_size.height);
                (image_size.width * ratio, image_size.height * ratio)
            }
            ContentFit::Fill => (bounds_size.width, bounds_size.height),
            // ... other modes
        };
        
        // Center the image
        let x = self.bounds.x + (bounds_size.width - width) / 2.0;
        let y = self.bounds.y + (bounds_size.height - height) / 2.0;
        
        Rectangle { x, y, width, height }
    }
    
    fn load_image_bytes(&self) -> Result<Vec<u8>, SliderAtlasError> {
        // TODO: Critical integration point
        // Need to access app.panes[self.pane_idx].img_cache.get_image_bytes(self.image_idx)
        // Options:
        // 1. Pass bytes directly to widget (increases memory)
        // 2. Add global accessor (unsafe/tricky)
        // 3. Redesign to have cache accessible in primitive prepare
        unimplemented!("Need to bridge with existing image loading")
    }
}

struct PreparedSliderImage {
    info: AtlasImageInfo,
    content_bounds: Rectangle,
    viewport: Viewport,
}
```

**Critical Integration Challenge**: Image Bytes Access

The `SliderImagePrimitive::prepare()` needs access to image bytes, but we're in a rendering context without direct access to `app.panes`. 

**Solution Options:**

**Option A: Pre-load bytes in async task (Recommended)**
```rust
// In navigation_slider.rs, modify create_async_image_widget_task
pub async fn create_async_slider_atlas_task(
    img_path: PathSource,
    pos: usize,
    pane_idx: usize,
    archive_cache: Option<Arc<Mutex<ArchiveCache>>>,
) -> Result<(usize, usize, Vec<u8>, (u32, u32)), (usize, usize)> {
    let bytes = /* load bytes */;
    let dimensions = /* extract dimensions */;
    
    // Don't create Handle - just return bytes
    Ok((pane_idx, pos, bytes, dimensions))
}

// In pane.rs
pub struct Pane {
    // ... existing fields
    pub slider_image_bytes: Option<Vec<u8>>,  // NEW: Store bytes for atlas upload
    pub slider_image_dimensions: Option<(u32, u32)>,
}

// Message handler stores bytes
Message::SliderImageAtlasLoaded(Ok((pane_idx, pos, bytes, dims))) => {
    let pane = &mut app.panes[pane_idx];
    pane.slider_image_bytes = Some(bytes);
    pane.slider_image_dimensions = Some(dims);
    Task::none()
}

// Widget gets bytes from pane
struct SliderImageShader {
    image_bytes: Option<Vec<u8>>,  // Passed from pane
    // ... other fields
}
```

**Option B: Store bytes in Storage (Alternative)**
```rust
// In prepare()
storage.store(SliderImageBytes {
    pane_idx: self.pane_idx,
    image_idx: self.image_idx,
    bytes: /* loaded earlier */,
});
```

**Recommendation**: Use Option A - cleaner separation, bytes loaded async and passed down through widget

**Testing:**
- Visual test: Render slider widget → verify image appears correctly
- Visual test: Move slider → verify smooth updates
- Visual test: Compare with existing Viewer widget → should look identical
- Integration test: Load large image (>2048px) → verify renders correctly

**Success Criteria:**
- Widget renders correctly
- Performance matches existing Viewer widget
- No visual differences compared to current slider rendering

---

### Phase 5: Message Handler Integration
**Goal**: Modify message handlers to use new atlas-based path

**Files to Modify:**
- `src/app/message_handlers.rs`
- `src/navigation_slider.rs`
- `src/pane.rs`

**Changes to message_handlers.rs:**

```rust
// Modify handle_slider_messages
pub fn handle_slider_messages(app: &mut DataViewer, message: Message) -> Task<Message> {
    match message {
        Message::SliderChanged(pane_index, value) => {
            app.is_slider_moving = true;
            app.last_slider_update = Instant::now();
            
            // NEW: Use atlas-based loading instead of Handle-based
            navigation_slider::update_pos_atlas(
                &mut app.panes,
                pane_index,
                value as usize,
                true, // use_async
                false, // use_throttle
            )
        }
        
        // NEW: Handle atlas-based image loading
        Message::SliderImageAtlasLoaded(result) => {
            match result {
                Ok((pane_idx, pos, bytes, dimensions)) => {
                    let pane = &mut app.panes[pane_idx];
                    pane.slider_image_bytes = Some(bytes);
                    pane.slider_image_dimensions = Some(dimensions);
                    debug!("Atlas image loaded: pane={}, pos={}, dims={:?}", pane_idx, pos, dimensions);
                }
                Err((pane_idx, pos)) => {
                    warn!("Failed to load atlas image: pane={}, pos={}", pane_idx, pos);
                }
            }
            Task::none()
        }
        
        // Keep existing SliderReleased logic
        Message::SliderReleased(pane_index, value) => {
            app.is_slider_moving = false;
            // ... rest unchanged
        }
        _ => Task::none()
    }
}
```

**Changes to navigation_slider.rs:**

```rust
// NEW: Atlas-based loading task
pub async fn create_async_slider_atlas_task(
    img_path: PathSource,
    pos: usize,
    pane_idx: usize,
    archive_cache: Option<Arc<Mutex<ArchiveCache>>>,
) -> Result<(usize, usize, Vec<u8>, (u32, u32)), (usize, usize)> {
    let task_start = Instant::now();
    
    // Load bytes
    let bytes = match &img_path {
        PathSource::Filesystem(path) => std::fs::read(path),
        PathSource::Archive(_) | PathSource::Preloaded(_) => {
            if let Some(cache_arc) = archive_cache {
                let mut cache = cache_arc.lock().unwrap();
                crate::file_io::read_image_bytes(&img_path, Some(&mut *cache))
            } else {
                Err(std::io::Error::new(std::io::ErrorKind::InvalidInput, "Archive cache required"))
            }
        }
    }.map_err(|_| (pane_idx, pos))?;
    
    // Extract dimensions (for UI layout before atlas upload)
    let dimensions = image::load_from_memory(&bytes)
        .map(|img| (img.width(), img.height()))
        .map_err(|_| (pane_idx, pos))?;
    
    let total_time = task_start.elapsed();
    trace!("PERF: Atlas task time for pos {}: {:?}", pos, total_time);
    
    Ok((pane_idx, pos, bytes, dimensions))
}

// NEW: Atlas-based position update
pub fn update_pos_atlas(
    panes: &mut [pane::Pane],
    pane_index: isize,
    pos: usize,
    use_async: bool,
    use_throttle: bool,
) -> Task<Message> {
    // Same throttling logic as existing update_pos
    LATEST_SLIDER_POS.store(pos, Ordering::SeqCst);
    
    if use_throttle && !should_process_throttled() {
        return Task::none();
    }
    
    if use_async {
        let mut tasks = Vec::new();
        
        let pane_indices: Vec<usize> = if pane_index == -1 {
            panes.iter().enumerate()
                .filter_map(|(idx, pane)| if pane.dir_loaded { Some(idx) } else { None })
                .collect()
        } else {
            vec![pane_index as usize]
        };
        
        for idx in pane_indices {
            if let Some(pane) = panes.get(idx) {
                if pane.dir_loaded && pos < pane.img_cache.image_paths.len() {
                    let img_path = pane.img_cache.image_paths[pos].clone();
                    let archive_cache = if pane.has_compressed_file {
                        Some(Arc::clone(&pane.archive_cache))
                    } else {
                        None
                    };
                    
                    let task = Task::perform(
                        create_async_slider_atlas_task(img_path, pos, idx, archive_cache),
                        Message::SliderImageAtlasLoaded,
                    );
                    tasks.push(task);
                }
            }
        }
        
        return Task::batch(tasks);
    }
    
    Task::none()
}
```

**Changes to pane.rs:**

```rust
pub struct Pane {
    // ... existing fields
    
    // NEW: For atlas-based slider rendering
    pub slider_image_bytes: Option<Vec<u8>>,
    pub slider_image_dimensions: Option<(u32, u32)>,
    
    // DEPRECATED: Will remove after migration complete
    // pub slider_image: Option<Handle>,
}

impl Pane {
    pub fn build_ui_container(&self, is_slider_moving: bool, ...) -> Container<...> {
        if self.dir_loaded {
            if is_slider_moving && self.slider_image_bytes.is_some() {
                // NEW: Use SliderImageShader with atlas
                container(center(
                    SliderImageShader::new(
                        self.pane_id,
                        self.img_cache.current_index,
                        self.slider_image_bytes.clone().unwrap(), // TODO: Avoid clone
                    )
                    .width(Length::Fill)
                    .height(Length::Fill)
                    .content_fit(ContentFit::Contain)
                ))
                .width(Length::Fill)
                .height(Length::Fill)
            } else {
                // Existing ImageShader path unchanged
                // ...
            }
        }
    }
}
```

**Testing:**
- End-to-end test: Load directory → move slider → verify atlas rendering
- Performance test: Compare slider FPS (old vs new)
- Memory test: Monitor atlas memory usage during slider movement
- Edge case: Very large directory (1000+ images) → verify cache eviction

**Success Criteria:**
- Slider navigation works smoothly
- Performance matches or exceeds existing implementation
- No regressions in existing functionality

---

### Phase 6: Rendering Alignment Fix
**Goal**: Ensure pixel-perfect alignment between atlas and direct rendering

**The Root Cause:**

Visual jump happens because:
1. **Atlas path** (slider): Samples from atlas texture via shader with atlas coordinates
2. **Direct path** (normal): Samples from full texture via different shader

Even if they use the same `ContentFit::Contain` logic, sub-pixel differences accumulate:
- Atlas has texture coordinate transformation (world space → atlas space)
- Sampling might use different filtering (nearest vs linear)
- Vertex positions might round differently

**Solution: Unified Rendering Math**

Create shared shader module that both paths use:

```rust
// src/widgets/shader/unified_image_shader.wgsl
struct VertexOutput {
    @builtin(position) clip_position: vec4<f32>,
    @location(0) tex_coords: vec2<f32>,
};

struct ImageUniforms {
    // Transform from clip space to texture space
    image_transform: mat4x4<f32>,
    // Content fit parameters
    content_bounds: vec4<f32>,  // (x, y, width, height) in screen space
    viewport_size: vec2<f32>,
};

@group(0) @binding(0) var image_texture: texture_2d<f32>;
@group(0) @binding(1) var image_sampler: sampler;
@group(0) @binding(2) var<uniform> uniforms: ImageUniforms;

@vertex
fn vs_main(@builtin(vertex_index) vertex_index: u32) -> VertexOutput {
    var output: VertexOutput;
    
    // Generate full-screen quad
    let x = f32((vertex_index & 1u) << 2u);
    let y = f32((vertex_index & 2u) << 1u);
    
    // Transform to content bounds
    let scale = uniforms.viewport_size.xy;
    let pos = vec2<f32>(
        (uniforms.content_bounds.x + x * uniforms.content_bounds.z) / scale.x * 2.0 - 1.0,
        1.0 - (uniforms.content_bounds.y + y * uniforms.content_bounds.w) / scale.y * 2.0,
    );
    
    output.clip_position = vec4<f32>(pos, 0.0, 1.0);
    output.tex_coords = vec2<f32>(x, y);
    
    return output;
}

@fragment
fn fs_main(input: VertexOutput) -> @location(0) vec4<f32> {
    return textureSample(image_texture, image_sampler, input.tex_coords);
}
```

**Modify both TexturePipeline and AtlasTexturePipeline to use this shader:**

```rust
// In slider_atlas/pipeline.rs
impl AtlasTexturePipeline {
    pub fn render(
        &self,
        encoder: &mut wgpu::CommandEncoder,
        entry: &atlas::Entry,
        content_bounds: Rectangle,
        viewport: &Viewport,
        // ...
    ) {
        // Upload uniforms with EXACT same math as TexturePipeline
        let uniforms = ImageUniforms {
            image_transform: /* ... */,
            content_bounds: [
                content_bounds.x,
                content_bounds.y,
                content_bounds.width,
                content_bounds.height,
            ],
            viewport_size: [viewport.physical_width() as f32, viewport.physical_height() as f32],
        };
        
        // Upload to GPU
        queue.write_buffer(&self.uniform_buffer, 0, bytemuck::bytes_of(&uniforms));
        
        // Render with exact same pipeline setup as direct rendering
        // ...
    }
}
```

**Testing:**
- Visual test: Place horizontal/vertical ruler on screen → move slider → verify no jump
- Pixel-perfect test: Screenshot at slider release → compare pixels before/after
- COCO test: Load image with annotations → move slider → verify annotations stay aligned

**Success Criteria:**
- No visible jump when releasing slider
- Annotations remain pixel-perfect aligned
- Both rendering paths produce bit-identical output (where possible)

---

### Phase 7: Testing & Validation

**Unit Tests:**

```rust
#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_atlas_allocation() {
        let device = /* test device */;
        let mut atlas = Atlas::new(&device, wgpu::Backend::Vulkan, /* ... */);
        
        // Test single allocation
        let entry = atlas.upload(&device, &mut encoder, 512, 512, &dummy_data);
        assert!(entry.is_some());
        
        // Test allocation fills up and grows
        // ...
    }
    
    #[test]
    fn test_lru_eviction() {
        let mut cache = SliderAtlasCache::new(/* ... */, max_entries: 3);
        
        // Fill cache
        cache.get_or_upload(0, 0, &bytes_0);
        cache.get_or_upload(0, 1, &bytes_1);
        cache.get_or_upload(0, 2, &bytes_2);
        
        // Trigger eviction
        cache.get_or_upload(0, 3, &bytes_3);
        
        // Verify oldest (0, 0) was evicted
        assert!(!cache.entries.contains_key(&ImageKey { pane_idx: 0, image_idx: 0 }));
        assert!(cache.entries.contains_key(&ImageKey { pane_idx: 0, image_idx: 3 }));
    }
    
    #[test]
    fn test_cache_hit() {
        let mut cache = SliderAtlasCache::new(/* ... */);
        
        // Upload image
        let info1 = cache.get_or_upload(0, 5, &bytes).unwrap();
        
        // Retrieve same image
        let info2 = cache.get_or_upload(0, 5, &bytes).unwrap();
        
        // Should be same atlas entry
        assert_eq!(info1.entry, info2.entry);
    }
}
```

**Integration Tests:**

```rust
#[cfg(test)]
mod integration_tests {
    #[test]
    fn test_slider_navigation_atlas() {
        let mut app = create_test_app();
        load_test_directory(&mut app, "tests/fixtures/images");
        
        // Move slider
        app.update(Message::SliderChanged(-1, 5));
        
        // Verify atlas entry created
        let pane = &app.panes[0];
        assert!(pane.slider_image_bytes.is_some());
        
        // Release slider
        app.update(Message::SliderReleased(-1, 5));
        
        // Verify scene updated
        assert!(pane.scene.is_some());
    }
}
```

**Visual Tests:**

1. **Alignment Test**
   - Load image with high-contrast grid pattern
   - Move slider back and forth
   - Release at various positions
   - Verify no visible jump

2. **COCO Annotation Test**
   - Load COCO dataset image with annotations
   - Enable bounding boxes and masks
   - Move slider slowly
   - Verify annotations stay perfectly aligned

3. **Large Image Test**
   - Load 8192x8192 image
   - Verify atlas fragments correctly
   - Move slider
   - Verify no rendering artifacts

4. **Compression Test**
   - Enable BC1 compression
   - Load various image types (photos, graphics, text)
   - Verify quality acceptable
   - Compare file sizes vs memory usage

**Performance Benchmarks:**

```rust
#[bench]
fn bench_atlas_upload(b: &mut Bencher) {
    let mut cache = SliderAtlasCache::new(/* ... */);
    let bytes = load_test_image();
    
    b.iter(|| {
        cache.get_or_upload(0, 0, &bytes)
    });
}

#[bench]
fn bench_slider_navigation(b: &mut Bencher) {
    let mut app = create_test_app();
    load_test_directory(&mut app, "tests/fixtures/1000_images");
    
    b.iter(|| {
        for i in 0..100 {
            app.update(Message::SliderChanged(-1, i));
        }
    });
}
```

**Success Criteria:**
- All unit tests pass
- All integration tests pass
- Visual alignment test shows no jump
- Performance benchmarks show:
  - Atlas upload <5ms per image
  - Slider navigation >60fps
  - Cache hit rate >90%

---

## Risk Analysis & Mitigations

### Risk 1: Memory Usage Increase

**Description**: Atlas textures consume significant GPU memory
- 2048x2048 RGBA8 layer = 16MB
- 20 image cache might need multiple layers

**Mitigation:**
- Use BC1 compression (8:1 reduction) → 2MB per layer
- Tune `max_entries` based on available VRAM
- Add memory monitoring and warnings
- Implement aggressive eviction when memory pressure detected

**Monitoring:**
```rust
impl SliderAtlasCache {
    pub fn memory_usage_bytes(&self) -> u64 {
        let bytes_per_layer = match self.atlas.compression_strategy {
            CompressionStrategy::None => ATLAS_SIZE * ATLAS_SIZE * 4,
            CompressionStrategy::Bc1 => ATLAS_SIZE * ATLAS_SIZE / 2,
        };
        bytes_per_layer * self.atlas.layer_count() as u64
    }
}
```

### Risk 2: Atlas Fragmentation

**Description**: After many uploads/removals, atlas might fragment inefficiently

**Mitigation:**
- guillotiere allocator is proven robust (used in Firefox, Servo)
- Implement periodic defragmentation:
  ```rust
  pub fn defragment(&mut self) {
      if self.fragmentation_ratio() > 0.5 {
          // Copy all entries to new atlas
          // Drop old atlas
      }
  }
  ```
- For now, just monitor and warn

### Risk 3: BC1 Compression Quality

**Description**: BC1 lossy compression might cause visual artifacts

**Mitigation:**
- Make compression configurable (None/BC1)
- Use perceptual error metrics during encoding
- Add quality slider in settings
- Benchmark on diverse image types:
  - Photos: Usually good
  - Graphics/UI: May have artifacts
  - Text: Often poor (don't use BC1 for text)

### Risk 4: Shader Consistency

**Description**: Subtle shader differences could still cause alignment issues

**Mitigation:**
- Share exact shader code between paths
- Use uniform blocks with identical packing
- Add pixel-diff validation tests
- Extensive visual testing with grid patterns

### Risk 5: Integration Complexity

**Description**: Widget needs access to image bytes at render time

**Mitigation:**
- Pre-load bytes in async task (chosen solution)
- Store in `Pane` struct temporarily
- Clear after upload to atlas
- Document lifetime carefully

### Risk 6: Performance Regression

**Description**: Atlas upload might be slower than current approach

**Mitigation:**
- Benchmark against existing Viewer widget
- Profile upload path with `tracy`
- Optimize hot paths (e.g., use staging buffers)
- Consider async upload if needed

---

## Performance Considerations

### Upload Performance

**Current baseline** (iced Viewer widget):
- First load: ~2-5ms per image
- Cached: <0.1ms (atlas lookup)

**Target** (our atlas):
- First upload: <5ms per image (match current)
- Cached: <0.1ms (atlas lookup)

**Optimization strategies:**
- Use staging buffers for larger images
- Compress on background thread (if BC1 enabled)
- Batch uploads when slider moves rapidly

### Render Performance

**Target:** 60fps slider navigation

**Bottlenecks to watch:**
- Atlas texture binding (should be cached)
- Shader execution (very fast for simple sampling)
- CPU overhead in widget tree traversal

**Optimization:**
- Minimize state changes (reuse bind groups)
- Use push constants instead of uniform buffers (less overhead)
- Batch similar draw calls

### Memory Performance

**Atlas memory:**
- RGBA8: 16MB per 2048x2048 layer
- BC1: 2MB per layer
- Target: <100MB total for slider cache

**Monitoring:**
```rust
fn log_atlas_stats(cache: &SliderAtlasCache) {
    debug!("Atlas stats: {} entries, {} layers, {} MB",
           cache.entries.len(),
           cache.atlas.layer_count(),
           cache.memory_usage_bytes() / 1_000_000);
}
```

---

## Future Enhancements

### 1. Predictive Prefetching

Load images ahead of slider movement:
```rust
impl SliderAtlasCache {
    pub fn prefetch_adjacent(&mut self, pane_idx: usize, current_idx: usize, 
                             image_paths: &[PathSource], count: usize) {
        // Load next 'count' images in background
        // Load previous 'count' images in background
    }
}
```

### 2. Thumbnail Strip

Use atlas for fast thumbnail rendering:
```rust
pub struct ThumbnailStrip {
    pane_idx: usize,
    visible_range: Range<usize>,
    thumbnail_size: u32,
}

// Render many small images from atlas very efficiently
```

### 3. Image Comparison Mode

Side-by-side comparison using same atlas:
```rust
pub struct ComparisonView {
    left_image: (usize, usize),   // (pane_idx, image_idx)
    right_image: (usize, usize),
    // Both rendered from same atlas
}
```

### 4. Smart Cache Prioritization

Adjust cache priority based on user behavior:
```rust
struct CacheEntry {
    priority: f32,  // Higher = keep longer
    access_count: u32,
    last_access: Instant,
}

// Evict low-priority images first
```

### 5. Compression Strategy Selection

Per-image compression based on content:
```rust
fn select_compression(image: &Image) -> CompressionStrategy {
    if is_photo(image) {
        CompressionStrategy::Bc1  // Good for photos
    } else if has_text(image) {
        CompressionStrategy::None  // Text needs lossless
    } else {
        CompressionStrategy::Bc1
    }
}
```

---

## Conclusion

This integration plan provides a **comprehensive, phased approach** to bringing iced_wgpu's atlas rendering directly into viewskater. The result will be:

✅ **Consistent rendering**: No more visual jump between slider and normal modes  
✅ **Better performance**: Atlas caching proven by iced (fast slider navigation)  
✅ **More control**: Slider-optimized LRU, tailored for image viewer patterns  
✅ **Future-proof**: Foundation for thumbnails, comparison views, etc.  
✅ **Maintainable**: Self-contained in viewskater, no iced fork dependency  

### Implementation Timeline Estimate

- **Phase 1** (Atlas Module): 3-4 days
- **Phase 2** (Slider Cache): 2-3 days  
- **Phase 3** (Pipeline): 2-3 days
- **Phase 4** (Widget): 3-4 days
- **Phase 5** (Integration): 2-3 days
- **Phase 6** (Alignment Fix): 2-3 days
- **Phase 7** (Testing): 3-4 days

**Total: ~20-25 days** of focused development

### Next Steps

1. Review this plan thoroughly
2. Set up feature branch: `feature/slider-atlas-integration`
3. Begin Phase 1: Port atlas allocation code
4. Create test fixtures for validation
5. Iterate through phases, testing at each step

---

**Questions? Concerns? Ready to start?** 🚀

