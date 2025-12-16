# JPEG 2000 (.jp2) Support Plan

**Date:** 2025-12-14
**Status:** Implemented
**Feature Flag:** `jp2`

## Background

User request from Jan (Czech Republic) to support JPEG 2000 (.jp2) format for viewing space/satellite imagery. JPEG 2000 is commonly used for:
- Space/satellite imagery (NASA, ESA)
- Medical imaging (DICOM)
- Digital cinema (DCP packages)
- Archival/preservation (high quality, lossless option)

## Research Summary

### Image Crate Status
The main `image` crate does not have built-in JPEG 2000 support. A separate decoder is required.

### Available Rust Options

| Crate | Type | License | Status |
|-------|------|---------|--------|
| `jpeg2k` | High-level wrapper | MIT | Active (July 2025) |
| `openjpeg-sys` | C FFI bindings | BSD-2 | Stable |
| `openjp2` | Pure Rust port | BSD-2 | Experimental, active |
| `iszak/jpeg2000` | Pure Rust | MPL | Incomplete (no DWT/EBCOT) |

### Chosen Solution: `jpeg2k`

Repository: https://github.com/Neopallium/jpeg2k

**Reasons:**
- MIT licensed (wrapper), BSD-2 dependencies
- Compatible with ViewSkater's MIT/Apache-2.0 dual license
- Supports two backends:
  - `openjpeg-sys` (C FFI) - default, stable
  - `openjp2` (pure Rust) - optional, experimental
- Same maintainer for both backends (Neopallium)
- Integrates with `image::DynamicImage`

**Usage:**
```rust
let jp2_image = jpeg2k::Image::from_file("example.jp2")?;
let img: image::DynamicImage = jp2_image.try_into()?;
```

### Backend Options

```toml
# Default - uses C FFI (stable)
jpeg2k = "0.5"

# Pure Rust backend (experimental)
jpeg2k = { version = "0.5", default-features = false, features = ["openjp2"] }
```

Can start with C backend for stability, switch to pure Rust later without API changes.

## License Compatibility

| Component | License |
|-----------|---------|
| ViewSkater | MIT/Apache-2.0 |
| `jpeg2k` | MIT |
| `openjpeg-sys` / OpenJPEG | BSD-2 |
| `openjp2` | BSD-2 |

BSD-2 is more permissive than MIT/Apache-2.0. ViewSkater stays dual-licensed.

**Note:** When distributing binaries with `jp2` feature enabled, include BSD-2 copyright notice (e.g., in LICENSE-3RD-PARTY).

## Implementation (Completed)

### 1. Feature Flag and Dependency

**Cargo.toml:**
```toml
[features]
jp2 = ["dep:jpeg2k"]

[dependencies]
jpeg2k = { version = "0.10", optional = true, features = ["image"] }
```

Note: Version 0.10 is required for `image` crate 0.25 compatibility. The `image` feature enables `TryFrom<&Image> for DynamicImage`.

### 2. Allowed Extensions

**src/file_io.rs:**
```rust
#[cfg(feature = "jp2")]
const ALLOWED_EXTENSIONS_JP2: [&str; 3] = ["jp2", "j2k", "j2c"];
```

Added `is_supported_extension()` helper function to consolidate extension checking across the codebase.

### 3. JP2 Decode Path

**src/file_io.rs:**
```rust
/// Check if the given bytes represent a JPEG 2000 file by checking magic bytes
#[cfg(feature = "jp2")]
fn is_jp2_format(bytes: &[u8]) -> bool {
    // JP2 container: 0x0000000C 6A502020 0D0A870A
    // Raw codestream (j2k/j2c): 0xFF4FFF51
    ...
}

/// Decode JPEG 2000 image from bytes
#[cfg(feature = "jp2")]
fn decode_jp2(bytes: &[u8]) -> Result<DynamicImage, std::io::ErrorKind> {
    use jpeg2k::Image as Jp2Image;
    let jp2_image = Jp2Image::from_bytes(bytes)?;
    DynamicImage::try_from(&jp2_image)  // Note: TryFrom is for &Image, not Image
}

/// Unified decode function for all formats
pub fn decode_image_from_bytes(bytes: &[u8]) -> Result<DynamicImage, std::io::ErrorKind> {
    #[cfg(feature = "jp2")]
    if is_jp2_format(bytes) {
        return decode_jp2(bytes);
    }
    // Fall through to standard image crate
    ...
}
```

### 4. Files Modified

- `Cargo.toml` - Added `jpeg2k` dependency and `jp2` feature
- `src/file_io.rs` - Added JP2 extensions, magic byte detection, decode function, helper functions
- `src/cache/cache_utils.rs` - Updated `load_original_image` and `load_and_resize_image` to use unified `decode_image_from_bytes`

### 5. Testing

Tested with OpenJPEG conformance test file (`file1.jp2`):
- ✅ File loads and displays correctly
- ✅ Both `cargo build` and `cargo build --features jp2` compile
- ✅ Magic byte detection works for JP2 container format

## Future Considerations

- Switch to pure Rust `openjp2` backend once it stabilizes (same crate author)
- Consider JPEG XL support (separate feature, `jxl-oxide` crate exists)
- Add j2c/j2k raw codestream testing
