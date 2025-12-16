# Memory Warning Dialog Implementation

**Date:** 2025-09-06

## Overview

Implemented cross-platform memory warning dialog for large solid 7z archives to prevent system hangs and memory pressure, addressing contributor concerns from PR #52.

## Key Implementations

### 1. Memory Warning Dialog System
- **Native dialogs**: Using `rfd::MessageDialog` for cross-platform support (Windows/Linux/macOS)
- **Memory checking**: Added `check_memory_for_archive()` using `sysinfo` crate
- **User control**: Allows cancellation of large archive loading to prevent system instability
- **Configuration**: Configurable threshold via `CONFIG.archive_warning_threshold_mb = 500MB`

### 2. Integration with Existing Caching
- **Merged with contributor's smart caching**: Size-based preloading for ZIP/RAR (<200MB threshold)
- **Warning for solid 7z only**: Triggers for solid 7z files >500MB due to O(N) decompression penalty
- **Complementary systems**: Warning dialog works alongside existing per-pane archive caching

### 3. Bug Fixes Discovered
- **RAR preloading bug**: Fixed early `return Ok(())` that caused only first file to be preloaded
- **Loop optimization**: Removed unnecessary `enumerate()` calls in ZIP/RAR processing
- **M1 Mac memory issue**: Handled sysinfo bug where `available_memory()` returns 0.0GB on Apple Silicon

## Technical Details

### Performance Characteristics by Format
- **ZIP/RAR**: O(1) random access + smart caching based on total size
- **Non-solid 7z**: O(1) random access + lazy loading
- **Solid 7z**: O(N) sequential penalty → requires preloading or severe performance degradation

### Dialog Behavior
- **Ubuntu/Windows**: Shows accurate available memory info
- **macOS (M1)**: Gracefully handles sysinfo bug by hiding misleading 0.0GB display
- **Warning levels**: Info vs Warning based on available memory vs archive size ratio

## Discovered Issues for Future Work

### Preloading Logic Gap
Current implementation has architectural disconnect:
- **Initialization**: Preloads data into `OnceLock<Vec<u8>>` for small archives
- **Loading functions**: Always structured to call `archive_cache.read_compressed_file()` 
- **Reality**: Preloaded data accessed via `get_or_init()` but code path suggests archive cache usage

**Next steps**: Refactor to directly use preloaded data without archive cache fallback path for clarity.

## Files Modified

- `src/utils/mem.rs`: Added memory checking function
- `src/file_io.rs`: Added warning dialog function
- `src/config.rs`: Added configurable warning threshold
- `src/pane.rs`: Integrated dialog into 7z loading, fixed RAR bugs

## Commits Made

1. `feat: Add memory warning dialog for large solid 7z archives`
2. `fix: RAR preloading bug - only first file was being preloaded` 
3. `fix: Handle 0.0GB memory display when available_memory() fails`

## Performance Impact

- **Small archives**: No change (preloading works as before)
- **Large solid 7z**: User choice prevents system hangs
- **Cross-platform**: Consistent behavior with platform-specific workarounds

## Lessons Learned

- **Solid compression tradeoff**: Memory preloading vs sequential decompression performance
- **Platform differences**: macOS requires special handling for system memory APIs
- **Contributor collaboration**: Smart caching approach superior to all-or-nothing preloading
- **Code archaeology**: Found existing bugs during integration that improved overall robustness

