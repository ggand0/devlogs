# viewskater-egui: Minimal Feature Spec

**Goal:** Reproduce viewskater's core image rendering and async loading in egui. This serves as (1) an egui evaluation with a direct comparison point, and (2) the foundation for the LeRobot curation tool.

**Repo:** `viewskater-egui`

---

## Scope: What to Build

The minimal subset of viewskater that exercises all the patterns the curation tool will need. No COCO annotations, no archive support, no multi-platform packaging. Pure image viewing with caching.

---

## Feature Spec

### 1. Directory Loading

Open a directory (via CLI arg or drag-and-drop) and enumerate supported image files.

**ViewSkater reference:** `file_io.rs` (file enumeration, format filtering), `pane.rs` (directory loading)

**egui implementation:**
- CLI arg via `clap` (single positional path arg)
- `std::fs::read_dir` -> filter by supported extensions -> sort naturally
- Drag-and-drop via eframe's built-in `RawInput::dropped_files`
- Store as `Vec<PathBuf>`

**Supported formats (same as viewskater core):** jpg, png, bmp, webp, gif, tiff, qoi, tga

### 2. Image Display

Display the current image scaled to fit the window, maintaining aspect ratio.

**ViewSkater reference:** `ImageShader` widget with `ContentFit::Contain`, `TexturePipeline` for GPU rendering

**egui implementation:**
- Decode image to RGBA via `image` crate
- Upload via `ctx.load_texture()` -> get `TextureHandle`
- Display with `ui.image()` using `egui::load::SizedTexture`
- Content fit: calculate display rect manually to maintain aspect ratio within available space, or use egui's built-in image sizing with `fit_to_exact_size` / `max_size`
- No custom wgpu pipeline needed unless egui's image quality/perf is insufficient

### 3. Keyboard Navigation

Left/right arrow keys to navigate between images.

**ViewSkater reference:** `navigation_keyboard.rs` (continuous key-hold navigation with frame-rate gating)

**egui implementation:**
- Poll `ctx.input(|i| i.key_pressed(Key::ArrowRight))` each frame
- For continuous hold: check `ctx.input(|i| i.key_down(Key::ArrowRight))` with a frame-rate gate (e.g., navigate at most once per frame)
- On navigate: update `current_index`, display new image from cache or decode

### 4. Sliding Window Cache

Preload neighboring images in background threads so navigation is instant.

**ViewSkater reference:** `ImageCache` in `img_cache.rs` -- sliding window of `Vec<Option<CachedData>>` with `cache_count` neighbors on each side. `LoadOperation` enum for shift/load operations. `GpuImageCache` backend creates `wgpu::Texture` directly.

**egui implementation:**
- Cache structure: `Vec<Option<TextureHandle>>` of size `2 * cache_count + 1`
- Center slot = current image, slots on each side = preloaded neighbors
- Background loading via `std::thread::spawn` + `std::sync::mpsc::channel`:
  ```
  bg thread: receive (index, path) -> decode to RGBA -> send (index, ColorImage) back
  main thread: recv -> ctx.load_texture() -> store TextureHandle in cache slot
  ```
- On navigate forward: shift cache left, spawn load for new rightmost neighbor
- On navigate backward: shift cache right, spawn load for new leftmost neighbor
- On jump (slider): invalidate entire cache, reload centered on new position
- Default `cache_count`: 5 (same as viewskater)

**Key difference from viewskater:** viewskater's `GpuImageCache` creates `wgpu::Texture` objects directly on background threads (holding `Arc<Device>` + `Arc<Queue>`). In egui, texture upload must happen on the main thread via `ctx.load_texture()`. Background threads decode to `ColorImage` (CPU RGBA), main thread uploads. This is simpler but adds one frame of latency for the upload step. For the image sizes we care about, this is negligible.

### 5. Navigation Slider

A slider at the bottom to jump to any position in the directory.

**ViewSkater reference:** `navigation_slider.rs`, `DualSlider` custom widget

**egui implementation:**
- `egui::Slider` in a `TopBottomPanel::bottom()`
- Range: `0..=(num_files - 1)`
- On value change: jump `current_index`, invalidate cache, reload
- Display: `"42 / 1000"` label next to slider
- Drag behavior: egui's slider handles continuous drag natively (no atomics hack needed -- immediate mode means the slider value updates in the same frame)

### 6. Zoom and Pan

Mouse wheel zoom centered on cursor, click-drag to pan.

**ViewSkater reference:** `ImageShader` handles zoom/pan in the shader widget's `update()` method. Stores `scale` and `offset` in widget `State`. Uses NDC coordinate transforms in `TexturePipeline`.

**egui implementation:**
- Store `zoom: f32` and `pan: Vec2` in app state
- Mouse wheel: `ctx.input(|i| i.scroll_delta.y)` -> adjust zoom, recompute pan to keep cursor point fixed
- Drag: `response.drag_delta()` on the image area -> adjust pan
- Double-click to reset zoom/pan
- Apply transforms when computing the image display rect (scale size by zoom, offset by pan)
- If egui's built-in image widget can't handle subpixel positioning well enough, use `egui::PaintCallback` for raw wgpu rendering (same as viewskater's shader approach but without iced coupling)

### 7. Window Title

Show filename and position in the window title.

**ViewSkater reference:** `app.rs` `title()` method returns `"filename.jpg (42/1000) - ViewSkater"`

**egui implementation:**
- `ctx.send_viewport_cmd(egui::ViewportCommand::Title(...))`
- Format: `"filename.jpg (42/1000) - viewskater-egui"`

---

## What to Skip (not needed for evaluation)

| ViewSkater feature | Why skip |
|---|---|
| Multi-pane split view | Build later if egui evaluation passes (Phase 3) |
| Custom wgpu shader pipeline | Use egui's built-in image widget first; only add PaintCallback if quality/perf demands it |
| BC1 texture compression | Irrelevant for evaluation; RGBA is fine |
| Archive/ZIP support | Unrelated to framework evaluation |
| COCO annotation overlay | ViewSkater-specific feature |
| macOS sandbox / file bookmarks | Platform-specific, not framework-related |
| EXIF auto-rotation | Nice-to-have, add after core works |
| Settings file / config persistence | Start with hardcoded defaults |
| Replay/benchmark mode | Add later for perf comparison |
| Custom event loop / event batching | The whole point of egui is to not need this |
| Loading spinner | egui's immediate mode makes this trivial when needed |

---

## Architecture Overview

```
src/
  main.rs          -- eframe::run_native(), App struct, update() loop
  cache.rs         -- SlidingWindowCache, background loader thread
  file_io.rs       -- directory enumeration, format filtering
```

Three files. That's it for the MVP.

### App State

```rust
struct App {
    // Directory
    image_paths: Vec<PathBuf>,
    current_index: usize,

    // Cache
    cache: SlidingWindowCache,

    // View
    zoom: f32,
    pan: Vec2,
}
```

### SlidingWindowCache

```rust
struct SlidingWindowCache {
    textures: Vec<Option<TextureHandle>>,  // 2*cache_count+1 slots
    indices: Vec<isize>,                    // which image index each slot holds (-1 = empty)
    cache_count: usize,                     // how many neighbors to preload on each side

    // Background loading
    sender: Sender<LoadRequest>,            // send decode requests to bg thread
    receiver: Receiver<LoadResult>,         // receive decoded images from bg thread
}

struct LoadRequest {
    slot: usize,           // which cache slot this is for
    index: usize,          // image index in the directory
    path: PathBuf,
}

struct LoadResult {
    slot: usize,
    index: usize,
    image: ColorImage,     // decoded RGBA, ready for ctx.load_texture()
}
```

### Main Loop (inside `eframe::App::update`)

```rust
fn update(&mut self, ctx: &egui::Context, _frame: &mut eframe::Frame) {
    // 1. Poll background loader, upload completed images as textures
    self.cache.poll_loaded(ctx);

    // 2. Handle keyboard navigation
    if ctx.input(|i| i.key_pressed(Key::ArrowRight)) {
        self.navigate(1);
    }
    if ctx.input(|i| i.key_pressed(Key::ArrowLeft)) {
        self.navigate(-1);
    }

    // 3. Bottom panel with slider
    egui::TopBottomPanel::bottom("nav").show(ctx, |ui| {
        let mut idx = self.current_index as f64;
        if ui.add(egui::Slider::new(&mut idx, 0.0..=((self.image_paths.len()-1) as f64))).changed() {
            self.jump_to(idx as usize);
        }
        ui.label(format!("{} / {}", self.current_index + 1, self.image_paths.len()));
    });

    // 4. Central panel with image
    egui::CentralPanel::default().show(ctx, |ui| {
        if let Some(tex) = self.cache.current_texture() {
            // Apply zoom/pan, display image
            let size = tex.size_vec2() * self.zoom;
            let response = ui.add(
                egui::Image::from_texture(SizedTexture::new(tex.id(), size))
            );
            // Handle drag-to-pan, scroll-to-zoom on response
        }
    });
}
```

---

## Crate Dependencies

```toml
[dependencies]
eframe = "0.31"            # egui + winit + wgpu integration
egui = "0.31"
image = { version = "0.25", default-features = false, features = ["jpeg", "png", "bmp", "webp", "gif", "tiff", "qoi", "tga"] }
clap = { version = "4", features = ["derive"] }
log = "0.4"
env_logger = "0.11"
```

Six dependencies. Compare to viewskater's Cargo.toml.

---

## Success Criteria

The evaluation passes if:

1. **Navigation feels instant** with cache_count=5 on a directory of 100+ images (comparable to viewskater's keyboard nav speed)
2. **Slider drag is smooth** -- no stutter, no frame drops (the problem viewskater solved with atomics bypass shouldn't exist in immediate mode)
3. **Zoom/pan is responsive** -- no visible lag on scroll-to-zoom or drag-to-pan
4. **Code is simple** -- the entire app should be under 500 lines for the MVP. If it's approaching viewskater's complexity to achieve the same result, egui isn't winning.
5. **Total time to working prototype** -- should be achievable in a day or two, not a week

If any of these fail, document why and what would be needed to fix it. That data is valuable regardless of the framework decision.

---

## Performance Comparison Points

Once the MVP works, run the same image directory through both viewskater and viewskater-egui and compare:

| Metric | How to measure |
|---|---|
| Keyboard nav FPS | Hold right arrow, count frames per second |
| Cache hit latency | Time from keypress to new image displayed (should be <1 frame = <16ms) |
| Cache miss latency | Time to decode + upload a 4K image on cache miss |
| Slider drag FPS | Drag slider continuously, measure frame rate |
| Memory usage | RSS with cache_count=5 on the same directory |
| Startup time | Time from launch to first image displayed |
| Code size | Total lines of Rust (excluding comments/blanks) |
