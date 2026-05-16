# 030 Content-Based Image Format Detection

## PR #21: Image format decoding based on content, not on file name (by BafDyce)

PR replaces all `image::open()` calls with a content-based format detection approach using `ImageReader::open().with_guessed_format().decode()`. This fixes image loading failures when a file's extension doesn't match its actual format (e.g., a WebP image saved as `.jpg`).

### Problem

`image::open(path)` determines the image format solely from the file extension via `ImageFormat::from_path()`, which is a hardcoded extension-to-format map (`.jpg` → Jpeg, `.webp` → WebP, etc.). If the extension is wrong, the wrong decoder runs on the file data, causing a decode error. The image silently fails to load, and the user hits a dead end when navigating — they cannot navigate to or past the broken image, with no error message.

This also fails for files with no extension or non-standard extensions (e.g. `.jfif`), since `from_path` returns `Err(Unsupported)` immediately without even attempting to read the file.

### How `image::open()` works (extension only)

```
image::open(path)
  → ImageFormat::from_path(path)?        // extension string match: ".jpg" → Jpeg
  → load(BufReader::new(File::open()), format)  // decode with THAT format
```

No file content is ever inspected. A mismatched extension means the wrong decoder is invoked.

### How `ImageReader` + `with_guessed_format()` works (content fallback)

```
ImageReader::open(path)
  → format = ImageFormat::from_path(path).ok()   // extension check, stored as Option (None if unknown)

.with_guessed_format()
  → reads first 16 bytes of file
  → matches against MAGIC_BYTES table (23 entries: PNG header, JPEG 0xFF 0xD8 0xFF, "RIFF" for WebP, TIFF byte order marks, etc.)
  → if magic bytes match: REPLACES the extension-based format
  → if no match: KEEPS extension-based format as fallback

.decode()
  → decodes using the final determined format
```

The key line in `with_guessed_format()` (reader.rs:192):
```rust
self.format = format.or(self.format);
```

Content-based guess takes priority when it succeeds; extension is fallback only.

### Changes

- **`src/file_io.rs`**: New `open_image()` convenience wrapper:
  ```rust
  pub fn open_image(path: &Path) -> ImageResult<DynamicImage> {
      ImageReader::open(path)?.with_guessed_format()?.decode()
  }
  ```
- **`src/cache.rs`**: Two `image::open()` call sites replaced (background thread decode in `SlidingWindowCache`, sync decode in `decode_sync`)
- **`src/pane.rs`**: One `image::open()` call site replaced (initial image load in `load_image`)

Also includes incidental rustfmt reformatting (multi-line function signatures, trailing comma alignment, newline at EOF).

### Testing approach

The risk surface is narrow — a single code path (`guess_format_impl` matching `MAGIC_BYTES`) handles all formats, so minimal targeted tests cover it:

1. **Mismatched extension**: Take a real PNG, rename to `.jpg` → should load correctly (proves the fix)
2. **Correct extension**: Open a normal image directory → no regressions
3. **No extension**: Rename an image to have no extension → should load if magic bytes match
4. **Corrupt file**: Random data renamed to `.png` → should fail gracefully (warning logged, image skipped, no crash)

### Performance

No concern. The only overhead from `with_guessed_format()` is reading 16 bytes from an already-buffered `BufReader` and a linear scan of 23 small magic byte prefixes, then seeking back. Nanoseconds vs the milliseconds spent on actual image decoding. Both paths call the same `free_functions::load_inner()` for the decode itself.

### Review and merge

- Clippy: zero warnings
- All three `image::open()` call sites migrated; no remaining usages in the codebase
- Tested with a mixed directory: PNGs renamed to `.jpg`, JPG renamed to `.png` — all loaded correctly, no performance regressions
- Rustfmt changes accepted (planning to adopt it)
- File discovery settings toggles deferred to a separate PR
- Merged 2025-05-16

### Contributor's notes

- Apologized for rustfmt formatting noise (editor auto-formatted on save)
- Mentioned adding two settings toggles for file discovery options on his local branch, but they only take effect after app restart — invited to submit as separate PR
