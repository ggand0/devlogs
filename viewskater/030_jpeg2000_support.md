# JPEG 2000 Support Implementation

**Date:** 2024-12-14
**Branch:** `feat/jp2-support`
**Feature Flag:** `jp2`

## Summary

Added optional JPEG 2000 (.jp2, .j2k, .j2c) format support behind a feature flag. This was requested by a user (Jan from Czech Republic) who wanted to view space/satellite imagery.

## Changes

### Cargo.toml
- Added `jpeg2k` v0.10 as optional dependency with `image` feature
- Added `jp2` feature flag: `jp2 = ["dep:jpeg2k"]`

### src/file_io.rs
- Added `ALLOWED_EXTENSIONS_JP2` constant for jp2/j2k/j2c extensions
- Added `is_jp2_format()` - magic byte detection for JP2 container and raw codestream
- Added `decode_jp2()` - decodes JP2 bytes to `DynamicImage`
- Added `decode_image_from_bytes()` - unified decode function that handles both standard formats and JP2
- Added `is_supported_extension()` - helper to check extensions (consolidates checks)
- Updated `supported_image()` to include JP2 extensions when feature enabled
- Updated `pick_file()` to include JP2 extensions in file dialog filter
- Updated all extension checks to use `is_supported_extension()`

### src/cache/cache_utils.rs
- Updated `load_original_image()` to use `decode_image_from_bytes()`
- Updated `load_and_resize_image()` to use `decode_image_from_bytes()`
- Removed unused `Cursor` and `ImageReader` imports

## Technical Details

### Magic Byte Detection
- JP2 container format: `0x0000000C 6A502020 0D0A870A` (12 bytes)
- Raw JPEG 2000 codestream (j2k/j2c): `0xFF4FFF51` (4 bytes)

### jpeg2k Crate
- Version 0.10 required for `image` crate 0.25 compatibility
- Uses `openjpeg-sys` (C FFI) by default, can switch to `openjp2` (pure Rust) in future
- `TryFrom<&Image>` (not `TryFrom<Image>`) for DynamicImage conversion

### License
- `jpeg2k`: MIT
- `openjpeg-sys`/OpenJPEG: BSD-2
- Compatible with ViewSkater's MIT/Apache-2.0 dual license
- BSD-2 notice required when distributing binaries with `jp2` feature enabled

## Usage

```bash
# Build without JP2 support (default)
cargo build

# Build with JP2 support
cargo build --features jp2

# Run with JP2 support
cargo run --features jp2 -- /path/to/image.jp2
```

## Testing

Tested with multiple datasets:

1. **OpenJPEG conformance test files** (41 files, 22MB)
   - `/tmp/jp2_test_dataset/`
   - ~50% failed with SYCC colorspace or null pointer errors (intentionally malformed edge-case files)

2. **Sentinel-2 satellite imagery** (16 files, 774MB)
   - `/tmp/sentinel2_jp2/`
   - Dark grayscale spectral bands, some decode errors on unusual formats

3. **HiRISE Mars imagery** (41 files, 1.5GB)
   - `/tmp/hirise_mars_jp2/`
   - IRB false color composites (10-85MB each)
   - All loaded successfully

## Known Issues

- **Large file loading**: UI freezes ("not responding") when loading large JP2 files (10-85MB) due to synchronous loading blocking the UI thread
- **Future improvement**: Consider async loading or progressive decoding (JP2 natively supports resolution levels and tiling)

## PR

Merged as PR #69: https://github.com/ggand0/viewskater/pull/69
