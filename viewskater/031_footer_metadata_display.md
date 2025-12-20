# Footer Image Metadata Display

**Date**: 2024-12-14
**Issue**: #40 (partial implementation)
**Branch**: `feat/footer-metadata`

## Overview

This implementation adds image resolution and file size display to the footer, matching Eye of GNOME (EoG) style layout:

```
┌─────────────────────────────────────────────────────────────────────┐
│ 1920x1080 pixels  2.5 MB                                    42/500 │
└─────────────────────────────────────────────────────────────────────┘
```

- **Left side**: Resolution and file size (e.g., `1920x1080 pixels  2.5 MB`)
- **Right side**: Image index counter (e.g., `42/500`)

## Problem

Two main issues needed to be solved:

1. **Dimensions**: `CachedData::width()`/`height()` methods decode CPU images each time they're called (expensive operation)
2. **File size**: Not stored anywhere in the application - only available during initial file load

## Solution

Store metadata in a parallel data structure alongside the cached image data, captured at load time.

### Architecture

```
ImageCache
├── cached_data: Vec<Option<CachedData>>      // Existing: image data
├── cached_metadata: Vec<Option<ImageMetadata>> // NEW: metadata
└── ...
```

The metadata vector mirrors the `cached_data` vector, with the same indexing and shift behavior.

## Implementation Details

### 1. ImageMetadata Struct (`src/cache/img_cache.rs`)

```rust
#[derive(Debug, Clone, Default)]
pub struct ImageMetadata {
    pub width: u32,
    pub height: u32,
    pub file_size: u64,
}

impl ImageMetadata {
    pub fn new(width: u32, height: u32, file_size: u64) -> Self {
        Self { width, height, file_size }
    }

    pub fn resolution_string(&self) -> String {
        format!("{}x{}", self.width, self.height)
    }

    pub fn file_size_string(&self) -> String {
        if self.file_size < 1024 {
            format!("{} B", self.file_size)
        } else if self.file_size < 1024 * 1024 {
            format!("{:.1} KB", self.file_size as f64 / 1024.0)
        } else {
            format!("{:.1} MB", self.file_size as f64 / (1024.0 * 1024.0))
        }
    }
}
```

### 2. ImageCache Updates (`src/cache/img_cache.rs`)

Added `cached_metadata` field and updated all methods that manipulate the cache:

- `new()`: Initialize metadata vector
- `shift_cache_left()` / `shift_cache_right()`: Shift metadata alongside image data
- `move_next()` / `move_prev()`: Accept and store new metadata
- `move_next_edge()` / `move_prev_edge()`: Shift metadata for edge operations
- `clear_cache()`: Clear metadata vector
- `get_initial_metadata()`: Get metadata for current index
- `set_cached_metadata()`: Set metadata at specific cache position

### 3. File Loading with Size Capture (`src/file_io.rs`)

Added helper function to read image bytes with file size:

```rust
pub fn read_image_bytes_with_size(
    path_source: &PathSource,
    path: &std::path::Path,
    archive_cache: Option<&ArchiveCache>,
    preloaded_data: Option<&[u8]>,
) -> Result<(Vec<u8>, u64), io::Error>
```

- **Filesystem**: Uses `metadata.len()` to get file size
- **Archive/Preloaded**: Uses `data.len() as u64`

### 4. Async Loading Functions (`src/file_io.rs`)

Updated return types to include metadata:

```rust
// Before
pub async fn load_image_cpu_async(...) -> Result<Option<CachedData>, ...>

// After
pub async fn load_image_cpu_async(...) -> Result<Option<(CachedData, ImageMetadata)>, ...>
```

Metadata is captured from the decoded image dimensions and file size.

### 5. Message Type Update (`src/app/message.rs`)

```rust
// Before
ImagesLoaded(Result<(Vec<Option<CachedData>>, Option<LoadOperation>), ...>)

// After
ImagesLoaded(Result<(Vec<Option<CachedData>>, Vec<Option<ImageMetadata>>, Option<LoadOperation>), ...>)
```

### 6. Pane Struct (`src/pane.rs`)

Added field to track current image metadata:

```rust
pub current_image_metadata: Option<ImageMetadata>
```

Updated in `Default`, `new()`, and `reset_state()`.

### 7. Loading Handler (`src/loading_handler.rs`)

Updated handler signatures to accept and store metadata:

```rust
pub fn handle_load_operation_all(
    panes: &mut [pane::Pane],
    loading_status: &mut LoadingStatus,
    pane_indices: &[usize],
    target_indices: &[Option<isize>],
    image_data: &[Option<CachedData>],
    metadata: &[Option<ImageMetadata>],  // NEW
    op: &LoadOperation,
    operation_type: LoadOperationType,
)
```

### 8. Footer UI (`src/ui.rs`)

Updated `get_footer()` function signature:

```rust
pub fn get_footer(
    footer_text: String,           // Right side: "42/500"
    metadata_text: Option<String>, // Left side: "1920x1080 pixels  2.5 MB"
    pane_index: usize,
    show_copy_buttons: bool,
    options: FooterOptions,
) -> Container<'static, Message, WinitTheme, Renderer>
```

Layout uses `horizontal_space()` to push the index to the right:

```rust
row![
    left_content,      // Metadata
    horizontal_space(), // Flexible space
    right_content      // Index + buttons
]
```

## Files Modified

| File | Changes |
|------|---------|
| `src/cache/img_cache.rs` | Added `ImageMetadata` struct, `cached_metadata` field, updated shift/move methods |
| `src/pane.rs` | Added `current_image_metadata` field |
| `src/file_io.rs` | Added `read_image_bytes_with_size()`, updated async load functions |
| `src/app/message.rs` | Updated `ImagesLoaded` message type |
| `src/loading_handler.rs` | Updated handler signatures for metadata |
| `src/app/message_handlers.rs` | Updated to unpack metadata from message |
| `src/ui.rs` | Updated `get_footer()` and all 4 call sites |

## Edge Cases Handled

- **While loading**: Shows only index count when metadata is `None`
- **Archives (ZIP)**: File size captured from extracted data length
- **Dual pane mode**: Both panes show independent metadata

## Additional Fixes

### Metadata Updates During Navigation

Metadata is now updated in all navigation scenarios:

1. **Initial directory load**: Metadata set from cache after `load_initial_images()`
2. **Keyboard navigation**: `render_next_image()` / `render_prev_image()` update metadata from cache
3. **Slider release**: `load_full_res_image()` captures and sets metadata
4. **Slider dragging**: Async `SliderImageWidgetLoaded` handler sets metadata

The `SliderImageWidgetLoaded` message was extended to include `file_size`:
```rust
// Before
SliderImageWidgetLoaded(Result<(usize, usize, Handle, (u32, u32)), ...>)

// After
SliderImageWidgetLoaded(Result<(usize, usize, Handle, (u32, u32), u64), ...>)
```

### Configurable File Size Units

Added `use_binary_size` setting to allow users to choose between:

- **Decimal units** (default): `KB`, `MB` with 1000 divisor (matches GNOME/macOS/Windows file managers)
- **Binary units**: `KiB`, `MiB` with 1024 divisor (matches `ls -lh` and traditional computing)

```rust
// src/cache/img_cache.rs
pub fn file_size_string(&self, use_binary: bool) -> String {
    let (divisor, kb_suffix, mb_suffix) = if use_binary {
        (1024.0, "KiB", "MiB")
    } else {
        (1000.0, "KB", "MB")
    };
    // ...
}
```

Setting in `settings.yaml`:
```yaml
# Use binary file size units (true = KiB/MiB like ls -lh, false = KB/MB like GNOME)
use_binary_size: false
```

## Performance Issues Identified

During implementation, three double-read performance issues were introduced and subsequently fixed:

### Issue 1: GPU Cache Initial Load (`gpu_img_cache.rs:114`)

In `load_initial_images()`, the code calls `read_image_bytes_with_size()` to get the file size, which reads the entire file into memory. Then `self.load_image()` reads the file again for actual image decoding.

```rust
// Line 114: Reads entire file just to get size
let file_size = match crate::file_io::read_image_bytes_with_size(path_source, archive_cache.as_deref_mut()) {
    Ok((_, size)) => size,  // Discards the bytes!
    Err(_) => 0,
};

// Line 120: Reads file again for image loading
match self.load_image(cache_index as usize, image_paths, compression_strategy, archive_cache.as_deref_mut())
```

### Issue 2: Slider Release GPU Path (`navigation_slider.rs:138`)

After the GPU texture is already loaded with dimensions available from `texture.size()`, the code reads the entire file again just to get the file size:

```rust
// Line 137-141: Texture size is already available
let tex_size = texture.size();
let file_size = match crate::file_io::read_image_bytes_with_size(&img_path, None) {
    Ok((_, size)) => size,  // Reads entire file just for size!
    Err(_) => 0,
};
```

### Issue 3: Slider Release CPU Path (`navigation_slider.rs:163`)

Similar pattern - reads file for metadata, then `load_image()` reads it again:

```rust
// Lines 163-173: Reads file for metadata
let metadata = match crate::file_io::read_image_bytes_with_size(&img_path, archive_cache) {
    Ok((bytes, file_size)) => {
        let (width, height) = match image::load_from_memory(&bytes) {
            Ok(img) => img.dimensions(),
            // ...
        };
        Some(ImageMetadata::new(width, height, file_size))
    },
    Err(_) => None,
};

// Line 183: Reads file again
match img_cache.load_image(pos, archive_cache)
```

### Solution

Added `get_file_size()` function to `file_io.rs` that efficiently gets file size without reading content:

- **Filesystem files**: Uses `std::fs::metadata()` which only reads the inode (O(1) operation)
- **Archive/Preloaded content**: Falls back to archive cache to get content length

This is used where only file size is needed and the image will be loaded separately.

### Additional Optimization: Header-Only Dimension Reading

This optimization addresses a **pre-existing inefficiency** in `CachedData::width()` and `height()` methods, which used `image::load_from_memory()` (full decode) to get dimensions.

Changed all `image::load_from_memory()` calls used for dimension extraction to use `ImageReader::into_dimensions()` instead. This reads only the image header (format-specific metadata) rather than fully decoding the image pixels.

**Locations optimized:**
- `CachedData::dimensions()` in `img_cache.rs` - Now uses header-only read (pre-existing inefficiency)
- `CachedData::width()` / `height()` - Simplified to use `dimensions()` (pre-existing inefficiency)
- `cpu_img_cache.rs:load_initial_images()` - Initial CPU cache loading (new code)
- `file_io.rs:load_image_cpu_async()` - Async CPU loading path (new code)
- `navigation_slider.rs:create_async_image_widget_task()` - Slider dragging async task (new code)

**Why this matters:**
- `image::load_from_memory()` fully decodes the image (decompresses all pixels) - O(width×height)
- `ImageReader::into_dimensions()` only parses the image header - typically O(1) or O(few bytes)

## Responsive Footer for Narrow Windows

In dual pane mode, the footer can become cramped when the window is narrow. Implemented progressive element hiding based on available width.

### Phases (7 levels)

| Phase | Metadata | Copy Buttons | Index Display |
|-------|----------|--------------|---------------|
| 1 | Resolution + file size (e.g., "1920x1080 pixels  2.5 MB") | ✓ | index/total |
| 2 | Resolution with "pixels" (e.g., "1920x1080 pixels") | ✓ | index/total |
| 3 | Dimensions only (e.g., "1920x1080") | ✓ | index/total |
| 4 | None | ✓ | index/total |
| 5 | None | ✗ | index/total |
| 6 | None | ✗ | index only |
| 7 | None | ✗ | (empty) |

### Implementation

**`ResponsiveFooterState` struct and `get_responsive_footer_state()` function (`src/ui.rs`):**

```rust
struct ResponsiveFooterState {
    metadata: Option<String>,
    show_copy_buttons: bool,
    footer_text: String,
}

fn get_responsive_footer_state(
    available_width: f32,
    metadata_text: &Option<String>,
    footer_text: &str,
    show_copy_buttons: bool,
) -> ResponsiveFooterState
```

Returns a state struct that controls:
- `metadata`: Full, resolution+pixels, dimensions-only, or None
- `show_copy_buttons`: Whether to render copy buttons
- `footer_text`: Full "index/total", index only, or empty

**Dynamic text measurement:**

Instead of using hardcoded character width approximations, the responsive footer uses actual font measurements via iced's `cosmic_text` font system:

```rust
fn measure_text_width(text: &str) -> f32 {
    let mut font_system_guard = font_system()
        .write()
        .expect("Failed to acquire font system lock");

    let mut buffer = cosmic_text::Buffer::new(
        font_system_guard.raw(),
        cosmic_text::Metrics::new(14.0, 14.0 * 1.3),
    );

    buffer.set_size(font_system_guard.raw(), Some(10000.0), Some(100.0));
    buffer.set_text(
        font_system_guard.raw(),
        text,
        to_attributes(Font::MONOSPACE),
        to_shaping(Shaping::Basic),
    );

    measure_buffer(&buffer).width
}
```

This creates a text buffer with the same font settings as the footer (Font::MONOSPACE at size 14) and returns the actual rendered width. This is more accurate than the previous fixed 8.5px character width estimate, which caused elements to disappear too early.

**Fixed width constants (for non-text elements):**
- Button width: 26px each (18px icon + padding)
- Button spacing: 3px
- Footer padding: 6px (3px each side)
- Element spacing: 3px
- Minimum margin: 5px (before hiding elements)

**Window width tracking:**

1. `window_width: f32` field added to `DataViewer` struct
2. `WindowResized(f32)` message added to `Message` enum
3. `main.rs` sends `WindowResized` on `WindowEvent::Resized`
4. Handler updates `app.window_width`

**macOS Retina display fix:**

`WindowEvent::Resized` provides physical pixels, but on macOS Retina displays the scale factor is typically 2.0. The width must be divided by `window.scale_factor()` to get logical pixels:

```rust
WindowEvent::Resized(size) => {
    let logical_width = size.width as f32 / window.scale_factor() as f32;
    state.queue_message(Message::WindowResized(logical_width));
}
```

Without this fix, macOS would report a 1920px window as 3840px, causing the responsive footer to think there's twice as much space available.

**Width calculation for dual pane:**

Each pane gets `window_width / 2.0` as available width for its footer.

### Files Modified

| File | Changes |
|------|---------|
| `src/ui.rs` | Added `ResponsiveFooterState` struct, `get_responsive_footer_state()` function, updated `get_footer()` to use state |
| `src/app.rs` | Added `window_width: f32` field |
| `src/app/message.rs` | Added `WindowResized(f32)` message |
| `src/app/message_handlers.rs` | Added handler for `WindowResized` |
| `src/main.rs` | Queue `WindowResized` on window resize events |

## Archive File Size Fix

When viewing images inside ZIP archives and using slider navigation, the file size was showing "0 B" on slider release.

**Root cause:** `get_file_size()` was called with `None` for the archive cache:

```rust
let file_size = crate::file_io::get_file_size(&img_path, None);
```

For `PathSource::Archive`, the function needs the archive cache to look up file sizes. Without it, it returns 0.

**Fix:** Pass the archive cache to `get_file_size()` in the slider release path (`navigation_slider.rs`). The fix gets the file size while still holding the archive lock (needed for the preceding image load), then explicitly drops the lock before continuing.

## Future Work

Issue #40 also mentions:
- EXIF data display
- Additional image properties
- Possible tool window for detailed metadata

This implementation provides the foundation for those features by establishing the metadata storage and display infrastructure.
