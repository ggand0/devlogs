# SliderEngine Integration into main.rs

**Date:** 2025-11-04

## Current Flow (main.rs:728-747)

```rust
let present_start = Instant::now();
{
    let mut engine_guard = engine.lock().unwrap();
    let mut renderer_guard = renderer.lock().unwrap();

    // 1. iced renders everything
    renderer_guard.present(
        &mut engine_guard,
        device,
        queue,
        &mut encoder,
        None,
        frame.texture.format(),
        &view,
        viewport,
        &debug_tool.overlay(),
    );

    // 2. Submit all GPU commands
    engine_guard.submit(queue, encoder);
}
```

**Problem:** Where does `SliderEngine` fit?

---

## NEW Flow with SliderEngine

```rust
let present_start = Instant::now();
{
    // STEP 1: SliderEngine processes pending uploads (batched)
    {
        let mut slider_engine_guard = slider_engine.lock().unwrap();
        slider_engine_guard.prepare(
            device,
            queue,
            &mut encoder,
            app.slider_value as usize,  // Current position for priority
        );
    }
    
    // STEP 2: iced renders everything (including slider primitive)
    {
        let mut engine_guard = engine.lock().unwrap();
        let mut renderer_guard = renderer.lock().unwrap();

        renderer_guard.present(
            &mut engine_guard,
            device,
            queue,
            &mut encoder,  // SAME encoder - commands are accumulated
            None,
            frame.texture.format(),
            &view,
            viewport,
            &debug_tool.overlay(),
        );

        // STEP 3: Submit EVERYTHING in one batch
        engine_guard.submit(queue, encoder);
    }
}
```

---

## Detailed Explanation

### STEP 1: SliderEngine.prepare()
```rust
// In SliderEngine::prepare()
pub fn prepare(
    &mut self,
    device: &wgpu::Device,
    queue: &wgpu::Queue,
    encoder: &mut wgpu::CommandEncoder,
    current_position: usize,
) {
    // Process 1-2 pending uploads
    for _ in 0..MAX_UPLOADS_PER_FRAME {
        if let Some(upload) = self.dequeue_next(current_position) {
            // Upload to atlas (adds commands to encoder, DOESN'T submit)
            if let Some(entry) = self.atlas.upload(device, encoder, ...) {
                self.entries.insert(upload.key, entry);
                
                // Create GPU resources (buffers, bind groups)
                let resources = self.pipeline.create_resources(device, &entry);
                self.gpu_cache.put(upload.key, resources);
            }
        }
    }
    
    // Staging belt recall (like iced's engine does)
    self.staging_belt.recall();
}
```

**Key:** Commands are added to `encoder`, but **NOT submitted yet**.

### STEP 2: iced's renderer.present()
```rust
// Inside iced_wgpu's renderer
pub fn present(&mut self, engine: &mut Engine, ..., encoder: &mut CommandEncoder, ...) {
    // Render all widgets
    for primitive in primitives {
        match primitive {
            Primitive::Custom(slider_image) => {
                // Calls our SliderImagePrimitive::render()
                slider_image.render(encoder, target, ...);
            }
            // ... other primitives
        }
    }
}
```

**Key:** Slider primitive's `render()` draws using **already-uploaded** atlas data. No blocking.

### STEP 3: engine.submit()
```rust
// Inside iced's Engine::submit()
pub fn submit(&mut self, queue: &wgpu::Queue, encoder: CommandEncoder) {
    let index = queue.submit(Some(encoder.finish()));
    self.staging_belt.recall();
}
```

**Key:** ONE submit for all work - SliderEngine uploads + iced rendering.

---

## SliderImagePrimitive Changes

### OLD (Shader Widget Approach - BLOCKING)
```rust
impl shader::Primitive for SliderImagePrimitive {
    fn prepare(&self, device: &Device, queue: &Queue, ...) {
        let mut encoder = device.create_command_encoder(...);
        
        // Upload NOW - BLOCKS 26ms
        atlas.upload(device, &mut encoder, ...);
        queue.submit(Some(encoder.finish()));  // ❌ BLOCKS
    }
    
    fn render(&self, encoder: &mut CommandEncoder, ...) {
        // Draw the image
    }
}
```

### NEW (Engine Approach - NON-BLOCKING)
```rust
impl shader::Primitive for SliderImagePrimitive {
    fn prepare(&self, device: &Device, queue: &Queue, ..., storage: &mut Storage) {
        // Nothing! Upload already happened in SliderEngine::prepare()
        // Just verify resources exist
        let engine = storage.get::<SliderEngine>().unwrap();
        if !engine.has_resources(self.key) {
            debug!("Resources not ready yet for {:?}", self.key);
        }
    }
    
    fn render(&self, encoder: &mut CommandEncoder, target: &TextureView, ..., storage: &Storage) {
        let engine = storage.get::<SliderEngine>().unwrap();
        
        // Render using pre-uploaded resources
        if let Some(resources) = engine.get_resources(self.key) {
            engine.pipeline.render(resources, encoder, target, self.bounds);
        }
    }
}
```

---

## Data Structure in main.rs

```rust
pub struct DataViewer {
    // ... existing fields ...
    
    // iced's engine (existing)
    pub engine: Arc<Mutex<iced_wgpu::Engine>>,
    pub renderer: Arc<Mutex<Renderer>>,
    
    // NEW: SliderEngine
    pub slider_engine: Arc<Mutex<SliderEngine>>,
}
```

---

## Full Integration Example

```rust
// In main.rs event loop
Event::WindowEvent { event: WindowEvent::RedrawRequested, .. } => {
    let frame = surface.get_current_texture().unwrap();
    let view = frame.texture.create_view(&Default::default());
    
    let mut encoder = device.create_command_encoder(&CommandEncoderDescriptor {
        label: Some("Render Encoder"),
    });
    
    let present_start = Instant::now();
    
    // ========================================
    // STEP 1: SliderEngine processes uploads
    // ========================================
    {
        let mut slider_engine = app.slider_engine.lock().unwrap();
        
        slider_engine.prepare(
            &device,
            &queue,
            &mut encoder,
            app.slider_value as usize,
        );
        
        // Timing
        let slider_prep_time = present_start.elapsed();
        if slider_prep_time.as_millis() > 20 {
            debug!("SliderEngine prepare took {}ms", slider_prep_time.as_millis());
        }
    }
    
    // ========================================
    // STEP 2: iced renders everything
    // ========================================
    {
        let mut engine_guard = app.engine.lock().unwrap();
        let mut renderer_guard = app.renderer.lock().unwrap();

        renderer_guard.present(
            &mut engine_guard,
            &device,
            &queue,
            &mut encoder,
            None,
            frame.texture.format(),
            &view,
            &viewport,
            &debug.overlay(),
        );

        // ========================================
        // STEP 3: Submit everything ONCE
        // ========================================
        engine_guard.submit(&queue, encoder);
    }
    
    frame.present();
    
    let present_time = present_start.elapsed();
    if present_time.as_millis() > 16 {
        warn!("Frame took {}ms", present_time.as_millis());
    }
}
```

---

## Key Advantages

| Aspect | OLD (Shader Widget) | NEW (SliderEngine) |
|--------|---------------------|-------------------|
| Uploads per frame | All queued | 1-2 (configurable) |
| Blocking | Yes (26ms) | No |
| Submit calls | 1 per upload | 1 per frame total |
| Frame budget | Unpredictable | Controlled |
| Integration | Widget-level | Engine-level |

---

## Message Handler Integration

```rust
// In message_handlers.rs
Message::SliderImageLoaded(pane_idx, pos, rgba_bytes, dims) => {
    // Queue upload (instant, non-blocking)
    let key = ImageKey { pane_idx, image_idx: pos };
    
    let mut slider_engine = app.slider_engine.lock().unwrap();
    slider_engine.queue_upload(key, Arc::new(rgba_bytes), dims);
    
    Task::none()
}
```

**No blocking!** Upload queued instantly, processed later in main loop.

---

## Performance Expectations

### Before (Shader Widget)
```
User moves slider: 10 → 50
Frame 1: Upload pos 50 → BLOCKS 26ms → render
Frame 2: Upload pos 49 → BLOCKS 26ms → render
...
Result: 30 FPS, laggy
```

### After (SliderEngine)
```
User moves slider: 10 → 50
Frame 1: Queue 40 uploads → 0ms
Frame 2: Prepare pos 50 (priority) → 26ms → render pos 50
Frame 3: Prepare pos 49 → 26ms → render pos 49
Frame 4: Prepare pos 48 → 26ms → render pos 48
...

Cached images (pos 10-20): render immediately → 120+ FPS
```

**Current position renders in 1-2 frames, others load in background without lag.**

---

## Questions?

1. Does this integration flow make sense?
2. Any concerns about the architecture?
3. Ready to proceed with implementation?

