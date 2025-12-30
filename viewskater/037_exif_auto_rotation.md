# EXIF Auto-Rotation Implementation

**Date:** 2025-12-30
**Branch:** `feat/exif-rotation`
**Commit:** `3b2a479`

## Summary

Added automatic EXIF orientation correction so images (primarily JPEGs from smartphones/cameras) display in the correct orientation regardless of how the camera was held when the photo was taken.

## Problem

Digital cameras and smartphones store orientation metadata in EXIF tags rather than rotating the actual pixel data. Without EXIF orientation handling, images taken in portrait mode or at angles display incorrectly (rotated 90°, 180°, etc.).

## Solution

Used the `image` crate v0.25's built-in EXIF orientation support:
- `ImageDecoder::orientation()` - reads EXIF orientation tag
- `DynamicImage::apply_orientation()` - applies rotation/flip transformation

No additional dependencies required.

## EXIF Orientation Values

| Value | Transformation | Description |
|-------|---------------|-------------|
| 1 | None | Normal |
| 2 | FlipHorizontal | Mirrored |
| 3 | Rotate180 | Upside down |
| 4 | FlipVertical | Mirrored vertically |
| 5 | Rotate90FlipH | Transpose |
| 6 | Rotate90 | Rotated 90° CW |
| 7 | Rotate270FlipH | Transverse |
| 8 | Rotate270 | Rotated 270° CW |

## Files Changed

### New File: `src/exif_utils.rs`

Two functions:

```rust
/// Decodes image with EXIF orientation applied
pub fn decode_with_exif_orientation(bytes: &[u8]) -> Result<DynamicImage, std::io::ErrorKind>

/// Gets dimensions accounting for orientation (swaps w/h for 90/270 rotations)
pub fn get_orientation_aware_dimensions(bytes: &[u8]) -> (u32, u32)
```

### Modified Files

| File | Change |
|------|--------|
| `src/main.rs` | Register `exif_utils` module |
| `src/file_io.rs` | `decode_image_from_bytes()` now uses EXIF-aware decode |
| `src/cache/texture_cache.rs` | Use EXIF-aware decode for texture creation |
| `src/widgets/shader/cpu_scene.rs` | 3 locations updated to use EXIF-aware decode |
| `src/cache/img_cache.rs` | `CachedData::dimensions()` now accounts for orientation |

## Implementation Details

### Decode Flow

```
bytes
  ↓
ImageReader::with_guessed_format()
  ↓
into_decoder()
  ↓
decoder.orientation()  →  Get EXIF orientation (or NoTransforms)
  ↓
DynamicImage::from_decoder()
  ↓
img.apply_orientation()  →  Apply rotation/flip if needed
  ↓
Correctly oriented DynamicImage
```

### Fallback Handling

If `into_decoder()` fails (some formats don't support the decoder interface), falls back to simple decode without orientation correction.

### JPEG 2000 (JP2)

JP2 format doesn't use EXIF orientation, so it bypasses the EXIF-aware path and decodes directly.

### Dimension Calculation

For `CachedData::Cpu` (raw bytes), dimensions are calculated with orientation awareness:
- Orientations 5, 6, 7, 8 (90°/270° rotations) swap width and height
- Other orientations keep original dimensions

GPU textures (`CachedData::Gpu`, `CachedData::BC1`) report dimensions directly from the already-rotated texture.

## Testing

Tested with smartphone photos taken in various orientations. Images now display correctly matching the system image viewer.

## Performance

Minimal overhead:
- `orientation()` only reads EXIF metadata, not full image decode
- Decoder is reused for both orientation reading and image decoding
- Transformation only performed when orientation != NoTransforms
