# Compressed File Support PR #52

**Date:** 2025-09-04

## PR Overview

**Title**: Feature Proposal  
**PR Number**: #52  
**Author**: [@hml-pip](https://github.com/hml-pip)  
**Reviewer**: [@ggand0](https://github.com/ggand0)  
**Status**: Under Review  
**Branch**: `hml-pip:viewer` → `ggand0:main`  

## Summary

This PR adds comprehensive compressed file support (ZIP, RAR, 7z) and fullscreen mode functionality to ViewSkater. The implementation includes per-pane archive caching for optimal performance and dual-pane support.

## Key Features Added

### 1. Compressed File Support
- **ZIP files**: Full support with cached archive instances
- **RAR files**: Support with performance optimizations 
- **7z files**: Support for both solid and non-solid compression
- **Lazy loading**: For memory-efficient handling of large archives
- **Preloading**: For small archives (<500MB) with optimal performance

### 2. Fullscreen Mode
- **F11 key**: Toggle fullscreen mode
- **Escape key**: Exit fullscreen mode
- **Cross-platform**: Proper handling for Windows, Linux, and macOS
- **macOS fixes**: Correct use of `set_fullscreen()` vs `set_simple_fullscreen()`
- **UI adjustments**: Menu behavior optimizations for fullscreen

### 3. Mouse Navigation Enhancements
- **Mouse wheel navigation**: Navigate between images
- **Ctrl/Cmd + wheel**: Zoom functionality
- **Smart behavior**: Context-aware mouse interactions

### 4. Performance Optimizations
- **Per-pane archive caching**: Each pane manages its own archive cache
- **Eliminated global state**: Removed global `COMPRESSED_FILE` variable
- **Archive instance reuse**: Cached ZIP/7z readers for efficiency
- **Debug logging optimization**: Removed expensive logging in RAR loops

## Technical Implementation

### Architecture Changes

#### Archive Cache System
```rust
pub struct ArchiveCache {
    current_archive: Option<(PathBuf, ArchiveType)>,
    zip_archive: Option<Arc<Mutex<zip::ZipArchive<BufReader<File>>>>>,
    sevenz_archive: Option<Arc<Mutex<sevenz_rust2::ArchiveReader<File>>>>,
}
```

#### Pane Integration
```rust
pub struct Pane {
    // ... existing fields ...
    pub archive_cache: Arc<Mutex<ArchiveCache>>,
    pub has_compressed_file: bool,
    pub mouse_wheel_zoom: bool,
    pub ctrl_pressed: bool,
}
```

#### Path Type Enum
```rust
pub enum PathType {
    PathBuf(PathBuf),
    FileByte(String, std::sync::OnceLock<bytes::Bytes>)
}
```

### Performance Results

| Archive Type | Before | After | Improvement |
|-------------|--------|-------|-------------|
| ZIP files | ~60 fps | ~60 fps | Maintained |
| RAR files | 25 fps | 60 fps | **140% improvement** |
| 7z files | 180-200 fps | 180-200 fps | Maintained |
| Regular directories | - | - | No impact |

### Memory Management Strategy

#### Small Archives (<500MB solid 7z)
- **Preload strategy**: All images loaded into memory during initialization
- **Performance**: 180-200 fps navigation
- **Memory usage**: High but manageable

#### Large Archives (≥500MB solid 7z)
- **Lazy loading strategy**: Images loaded on-demand
- **Performance**: Significantly slower due to solid compression format
- **Memory usage**: Low, only current image in memory
- **User warnings**: Clear notifications about performance trade-offs

#### Extreme Archives (≥2GB solid 7z)
- **Blocked loading**: Prevents system hangs
- **Error handling**: Graceful failure with user guidance
- **Recommendations**: Extract archive or use non-solid compression

## Conversation History

### Initial Feature Request (hml-pip - last month)
- Requested compressed file support and fullscreen mode
- Acknowledged uncertainty about performance impact
- Mentioned testing limitations (headless Mac Mini setup)

### Implementation Details (hml-pip - last month)

#### Compressed File Support
- Added support for ZIP, RAR, and 7z formats
- Implemented lazy loading for large files
- Plans for additional formats (7z confirmed working)
- Considered dedicated loading UI for large files

#### Fullscreen Window Mode
- Tested on Windows successfully
- Noted differences between macOS fullscreen methods
- Identified window title update issue after fullscreen toggle
- Made UI tweaks for fullscreen mode

#### Mouse Features
- Added mouse wheel navigation
- Implemented Ctrl/Cmd + wheel for zoom
- Enhanced menu interaction logic

### Performance Optimization Discoveries (hml-pip - 3 weeks ago)

#### Critical Performance Issues Found
1. **RAR Debug Logging**: Expensive `{:?}` formatting in tight loop
   ```rust
   // ❌ PERFORMANCE KILLER (25fps → 60fps impact)
   debug!("reading rar {} ?= {:?}", filename, entry_filename);
   ```

2. **Archive Re-parsing**: Every image access reopened archive files
3. **Global State Limitations**: Single global cache couldn't support dual-pane

#### Solutions Implemented
1. **Removed expensive debug logging** from RAR processing loops
2. **Implemented per-pane archive caching** for reusable archive instances
3. **Eliminated global state** in favor of proper encapsulation

### Testing and Platform Fixes (ggand0 - last week)

#### Linux Testing (Ubuntu 22.04)
- ✅ ZIP viewing: Working well
- ✅ Fullscreen mode: Working well
- ℹ️ Note: Fullscreen hides titlebar (expected behavior, same as browsers/eog)

#### macOS Testing (M1)
- ❌ ZIP viewing: Crashed on metadata files (e.g., `._` prefixes)
- ❌ Fullscreen: Used wrong winit method
- ✅ Fixed: Both issues resolved with targeted fixes

#### Fixes Applied
1. **macOS Metadata Filter**: 
   ```rust
   if name.starts_with("__MACOSX/") {
       return false;
   }
   ```

2. **macOS Fullscreen Correction**:
   - `set_simple_fullscreen()` → borderless window
   - `set_fullscreen()` → actual fullscreen

3. **Window Title Update Fix**:
   ```rust
   let new_title = state.program().title();
   if new_title != *last_title {
       window.set_title(&new_title);
       *last_title = new_title;
   }
   ```

### Advanced Optimizations (ggand0 - 4 days ago)

#### Per-Pane Archive Caching Implementation
- Achieved significant performance improvements
- Enabled dual-pane support for compressed files
- Removed vector cloning that caused slider lag

#### Performance Benchmarks
**Small Images Dataset Results (opt-dev profile, 7900XTX GPU):**
- Regular directory: 180-200 fps
- ZIP: 180-200 fps  
- RAR: 60+ fps
- Non-solid 7z: 110-120 fps
- Solid 7z: 180-200 fps

#### Remaining Issues Identified
1. **Slider lag with solid 7z**: Fixed by removing expensive vector clones
2. **macOS crash with large solid 7z**: Related to sevenz-rust2 library limitations
3. **Memory pressure**: Need size-based fallback logic

### Memory Management Solutions (hml-pip - 3 days ago)

#### 7z Codec Performance Analysis
- **LZMA2 codec**: Best performance for solid files
- **Other codecs**: Poor performance without LZMA2
- **File size impact**: Tested various compression methods

#### Error Handling Improvements
```rust
match reader.read_to_end(&mut buffer) {
    Ok(_) => {
        let ol = OnceLock::new();
        let _ = ol.set(bytes::Bytes::from(buffer));
        sevenz_list.lock().unwrap().push(PathType::FileByte(entry.name().to_string(), ol));
    },
    Err(e) => {
        error!("Failed to read entry {}: {}", entry.name(), e);
    },
}
```

### Final Integration (ggand0 - yesterday)

#### Slider Performance Fix
- Removed expensive vector cloning in async task creation
- Fixed lag issues with solid 7z navigation

#### Architecture Simplification Proposal
```rust
// Suggested refactoring to move solid 7z data to ArchiveCache
// file_io::read_image_bytes() → ArchiveCache method
// PathType::FileByte → simplified with HashMap lookup
```

#### Hybrid Loading Strategy
```rust
// Memory threshold approach
const LARGE_ARCHIVE_THRESHOLD: u64 = 500_000_000;
if is_solid && file_size < LARGE_ARCHIVE_THRESHOLD {
    // Preload for performance
} else {
    // Lazy load for memory efficiency
}
```

### Current Status (hml-pip - yesterday)

#### Global State Refactoring
- Successfully moved global flags to local state
- Implemented proper state threading through API
- Enhanced error handling for compressed files

#### Configuration Considerations
- Windows: Registry-based file associations planned
- Cross-platform: YAML config file approach preferred
- Scope: Deferred to separate PR for maintainability

## Technical Challenges Solved

### 1. Solid 7z Performance Problem
**Issue**: Solid 7z compression requires sequential decompression from beginning to access any file N (O(N) complexity per access).

**Solutions**:
- **Small files**: Preload entire archive into memory
- **Large files**: Lazy loading with user warnings about performance
- **Extreme files**: Block loading to prevent system hangs

### 2. Cross-Platform Fullscreen
**Issue**: Different fullscreen behaviors across platforms.

**Solutions**:
- **Windows/Linux**: Standard winit fullscreen handling
- **macOS**: Correct winit method selection and Escape key support
- **Menu interaction**: Enhanced detection zones for fullscreen mode

### 3. Archive Instance Caching
**Issue**: Re-parsing archives on every file access caused severe performance degradation.

**Solutions**:
- **Per-pane caching**: Each pane maintains its own archive instances
- **Format-specific optimizations**: ZIP and 7z cached, RAR optimized differently
- **Memory management**: Automatic cache clearing when switching archives

### 4. Dual-Pane Support
**Issue**: Global state prevented multiple archives from being loaded simultaneously.

**Solutions**:
- **Per-pane architecture**: Independent archive caches per pane
- **Thread safety**: Proper Arc<Mutex<>> usage for shared state
- **Load balancing**: Intelligent archive cache distribution

## Files Modified

### Core Implementation
- **`src/main.rs`** - Added `mod archive_cache;`
- **`src/archive_cache.rs`** - New module with complete `ArchiveCache` implementation
- **`src/pane.rs`** - Added archive cache and compression file support
- **`src/file_io.rs`** - Updated async loading functions, removed global state

### Cache System
- **`src/cache/img_cache.rs`** - Updated `PathType::bytes()` and image loading
- **`src/cache/cpu_img_cache.rs`** - Added archive cache parameter threading
- **`src/cache/gpu_img_cache.rs`** - Added archive cache parameter threading
- **`src/cache/cache_utils.rs`** - Updated image loading functions

### UI and Navigation
- **`src/navigation_slider.rs`** - Updated slider navigation for compressed files
- **`src/widgets/`** - Enhanced fullscreen mode support

### Dependencies
- **`Cargo.toml`** - Added compression library dependencies:
  ```toml
  zip = "4"
  unrar = "0.5"
  sevenz-rust2 = "0.18"
  bytes = "1.0" # Added for memory management
  ```

## Key Insights and Lessons

### 1. Performance Profiling Critical
- Debug logging in tight loops can cause 140% performance degradation
- Always profile actual bottlenecks rather than making assumptions
- String formatting (`{:?}`) especially expensive in hot paths

### 2. Architecture Enables Features
- Per-component state management enables dual-pane support
- Proper encapsulation eliminates global state issues
- Clean separation allows format-specific optimizations

### 3. Compression Format Characteristics
- **ZIP**: Good random access, efficient caching
- **RAR**: Sequential processing required, but cacheable
- **7z non-solid**: Excellent random access and performance
- **7z solid**: Excellent compression, terrible random access

### 4. Cross-Platform Considerations
- Platform-specific file metadata handling required
- Fullscreen behavior varies significantly between platforms
- Testing on actual hardware crucial for platform-specific features

### 5. Memory vs Performance Trade-offs
- Small archives: Memory usage acceptable for performance gains
- Large archives: Performance degradation acceptable to prevent crashes
- Extreme archives: Blocking necessary to prevent system instability

## Future Considerations

### Short-term Improvements
1. **Progress indicators** for large archive loading
2. **Cancellation support** for long-running operations
3. **User configuration** for memory/performance thresholds
4. **Background threading** for archive operations

### Medium-term Features
1. **Streaming decompression** for large solid archives
2. **Partial archive caching** strategies
3. **Format conversion** suggestions for users
4. **Archive extraction** integration

### Long-term Architecture
1. **Plugin system** for compression formats
2. **Advanced caching** with LRU eviction
3. **Network archive** support
4. **Cloud storage** integration

## Conclusion

This PR represents a significant enhancement to ViewSkater's capabilities, adding comprehensive compressed file support while maintaining excellent performance characteristics. The implementation demonstrates thoughtful engineering trade-offs between memory usage and performance, with proper fallback strategies for edge cases.

The collaborative development process between @hml-pip and @ggand0 showcased excellent problem-solving, with issues identified and resolved through systematic testing and optimization. The final implementation provides a solid foundation for future compression format support while ensuring system stability across all platforms.

**Status**: Ready for merge pending final testing and any remaining minor adjustments.

**Impact**: Significantly expands ViewSkater's utility for users working with compressed image archives, a commonly requested feature that enables new workflows for image management and viewing.
