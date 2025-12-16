# Custom Slider Engine Architecture

**Date:** 2025-11-04
**Goal:** Build a specialized rendering engine for slider images that achieves viewer widget performance while supporting atlas-based rendering for 4k images.

---

## Problem Analysis

### Why Shader Widget Approach Failed
```
Frame Timeline (4k image):
├─ prepare() starts
│  ├─ atlas.upload() → 26ms (BLOCKS)
│  └─ create GPU resources → 0.12ms
├─ render() → 7ms
└─ TOTAL: 33ms = ~30 FPS maximum

Problem: queue.submit() in prepare() blocks the entire frame
```

### Why Viewer Widget Is Fast
```
Frame Timeline (any image):
├─ prepare() on ALL widgets → collects operations (0ms blocking)
├─ engine.present() → ONE queue.submit() for everything
└─ TOTAL: ~7ms = 140+ FPS

Key: Deferred batching of all GPU work
```

---

## Proposed Architecture: `SliderEngine`

### Core Concept
Create a **mini-engine** specifically for slider rendering that:
1. **Batches GPU uploads** across frames (like iced's engine)
2. **Manages its own staging belt** for async uploads
3. **Maintains atlas state** without blocking prepare()
4. **Integrates with iced's renderer** as a custom primitive

### File Structure
```
src/
├── slider_engine/
│   ├── mod.rs              # Public API
│   ├── engine.rs           # Core engine: batching, staging belt
│   ├── atlas.rs            # Atlas management (reuse from slider_atlas/)
│   ├── pipeline.rs         # Rendering pipeline (reuse from slider_atlas/)
│   ├── upload_queue.rs     # Async upload queue
│   └── cache.rs            # LRU cache for atlas entries
└── widgets/
    └── slider_image.rs     # Widget using SliderEngine
```

---

## Architecture Details

### 1. `SliderEngine` (Core)

```rust
pub struct SliderEngine {
    // Batching infrastructure (like iced::Engine)
    staging_belt: wgpu::util::StagingBelt,
    
    // Atlas system
    atlas: Atlas,
    entries: HashMap<ImageKey, Entry>,
    
    // Rendering
    pipeline: AtlasPipeline,
    
    // Upload queue (non-blocking)
    upload_queue: VecDeque<PendingUpload>,
    
    // Resource cache
    gpu_cache: LruCache<ImageKey, GpuResources>,
}

struct PendingUpload {
    key: ImageKey,
    rgba_bytes: Arc<Vec<u8>>,
    dimensions: (u32, u32),
    priority: u32,  // Current slider position = highest priority
}

impl SliderEngine {
    /// Called from message handler - NEVER blocks
    pub fn queue_upload(&mut self, key: ImageKey, bytes: Arc<Vec<u8>>, dims: (u32, u32)) {
        self.upload_queue.push_back(PendingUpload { key, rgba_bytes: bytes, dimensions: dims, priority: 0 });
    }
    
    /// Called once per frame in prepare() - processes uploads in batches
    pub fn prepare(&mut self, device: &Device, encoder: &mut CommandEncoder, current_pos: ImageKey) {
        // Process up to N uploads per frame (e.g., 2 uploads = ~50ms budget)
        const MAX_UPLOADS_PER_FRAME: usize = 2;
        
        // Prioritize current position
        self.upload_queue.sort_by_key(|u| if u.key == current_pos { 0 } else { u.priority });
        
        for _ in 0..MAX_UPLOADS_PER_FRAME {
            if let Some(upload) = self.upload_queue.pop_front() {
                // Upload to atlas (records commands to encoder, doesn't submit)
                if let Some(entry) = self.atlas.upload(device, encoder, upload.dimensions.0, upload.dimensions.1, &upload.rgba_bytes) {
                    self.entries.insert(upload.key, entry);
                }
            }
        }
        
        // Staging belt recall (like iced does)
        self.staging_belt.recall();
    }
    
    /// Called in render() - draws current image if available
    pub fn render(&self, key: ImageKey, encoder: &mut CommandEncoder, target: &TextureView, bounds: Rectangle) {
        if let Some(entry) = self.entries.get(&key) {
            if let Some(resources) = self.gpu_cache.get(&key) {
                self.pipeline.render(resources, encoder, target, bounds);
            }
        }
    }
}
```

### 2. Integration with iced

```rust
// In main.rs or app.rs
pub struct DataViewer {
    // ... existing fields ...
    slider_engine: SliderEngine,
}

// In update() message handler
Message::SliderImageLoaded(pane_idx, pos, rgba_bytes, dims) => {
    let key = ImageKey { pane_idx, image_idx: pos };
    
    // Queue upload (non-blocking, instant return)
    app.slider_engine.queue_upload(key, Arc::new(rgba_bytes), dims);
    
    Task::none()
}
```

### 3. Custom Primitive

```rust
pub struct SliderImagePrimitive {
    key: ImageKey,
    bounds: Rectangle,
}

impl shader::Primitive for SliderImagePrimitive {
    fn prepare(&self, device: &Device, queue: &Queue, ..., storage: &mut Storage) {
        // Get engine from storage
        let engine = storage.get_mut::<SliderEngine>().unwrap();
        
        let mut encoder = device.create_command_encoder(...);
        
        // Process pending uploads (batched, non-blocking)
        engine.prepare(device, &mut encoder, self.key);
        
        // Submit batched work
        queue.submit(Some(encoder.finish()));
        engine.staging_belt.recall();
    }
    
    fn render(&self, encoder: &mut CommandEncoder, target: &TextureView, ..., storage: &Storage) {
        let engine = storage.get::<SliderEngine>().unwrap();
        engine.render(self.key, encoder, target, self.bounds);
    }
}
```

---

## Key Innovations

### 1. **Upload Queue with Priority**
```
User moves slider: pos 10 → 50
Queue contains: [11, 12, 13, ..., 49]

Frame 1: Upload [50 (priority), 49] → 50ms
Frame 2: Upload [48, 47] → 50ms
Frame 3: Upload [46, 45] → 50ms

Result: Current image (50) loads immediately, others load in background
```

### 2. **Configurable Budget**
```rust
const MAX_UPLOAD_TIME_PER_FRAME_MS: f64 = 16.0;  // Target 60 FPS
const MAX_UPLOADS_PER_FRAME: usize = 1;  // For 4k images (~26ms each)

// Adaptive: skip uploads if frame budget exceeded
if last_frame_time > 16ms {
    skip uploads this frame
}
```

### 3. **Staging Belt for Efficiency**
```rust
// Like iced's Engine
self.staging_belt.write_buffer(
    encoder,
    &atlas_texture,
    offset,
    wgpu::BufferSize::new(size),
    device,
);
```

### 4. **Smart Cache Eviction**
```rust
// Evict based on distance from current slider position
cache.retain(|key, _| {
    let distance = key.image_idx.abs_diff(current_position);
    distance <= CACHE_RADIUS  // e.g., keep ±10 images
});
```

---

## Performance Targets

| Scenario | Target FPS | Strategy |
|----------|-----------|----------|
| Small images (<2048) | 120+ FPS | Contiguous atlas, instant upload |
| 4k images (cached) | 120+ FPS | Already in atlas, zero upload |
| 4k images (uncached, current) | 30-60 FPS | Priority upload, 1-2 frames delay |
| 4k images (background load) | No impact | Load 1-2 per frame in background |

---

## Implementation Phases

### Phase 1: Core Engine (2-3 hours)
- [ ] Create `SliderEngine` struct
- [ ] Implement upload queue
- [ ] Add staging belt integration
- [ ] Basic prepare()/render() methods

### Phase 2: Integration (1-2 hours)
- [ ] Integrate with message handlers
- [ ] Create custom primitive
- [ ] Wire up to UI

### Phase 3: Optimization (1-2 hours)
- [ ] Adaptive upload budget
- [ ] Priority queue tuning
- [ ] Cache eviction strategy
- [ ] Performance profiling

### Phase 4: Polish (1 hour)
- [ ] Remove old shader widget code
- [ ] Add debug logging
- [ ] Update documentation

**Total Estimate: 5-8 hours**

---

## Advantages Over Shader Widget

| Aspect | Shader Widget | Custom Engine |
|--------|--------------|---------------|
| GPU upload | Blocking in prepare() | Batched, non-blocking |
| Frame budget | 26ms blocked per image | Configurable, adaptive |
| Queue management | None (processes all) | Priority-based queue |
| Cache strategy | LRU only | Distance-based + LRU |
| Integration | Per-widget | Centralized |
| 4k performance | 30 FPS (blocking) | 60+ FPS (deferred) |

---

## Next Steps

1. **Review this architecture** - any concerns or suggestions?
2. **Start Phase 1** - implement core `SliderEngine`
3. **Test incrementally** - verify each phase works before proceeding

Should I proceed with implementation?

