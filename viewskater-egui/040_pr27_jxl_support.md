# PR #27: JPEG XL Support via jxl-oxide

PR: https://github.com/ggand0/viewskater-egui/pull/27
Author: YelovSK (Patrik Hampel)

## Summary

Adds JPEG XL (.jxl) decoding support using the `jxl-oxide` crate (pure Rust).
Uses the `image` crate's decoding hook API via `jxl_oxide::integration::register_image_decoding_hook()`,
registered once with `std::sync::Once`. Minimal code changes -- adds "jxl" to supported extensions,
file dialog filter, README, .desktop, and macOS Info.plist.

## Decode Performance Investigation

During testing with 1080p JXL images, keyboard skating FPS dropped to ~15, compared to
~60 with PNG. Added timing instrumentation to `open_image()` and `image_to_color_image()`
to isolate the bottleneck.

### In-app decode times (10 concurrent background threads)

From debug logs, 1080p (1920x1080) JXL images:
```
open_image "frame_0001_lossless.jxl" 1920x1080 fmt=None: decode=283.4ms
open_image "frame_0002_lossless.jxl" 1920x1080 fmt=None: decode=294.2ms
open_image "frame_0011_lossy.jxl"    1920x1080 fmt=None: decode=212.8ms
```

PNG baseline for comparison:
```
open_image "frame_0001.png" 1920x1080 fmt=Some(Png): decode=27.5ms
```

### Standalone benchmark (examples/bench_jxl.rs)

To isolate whether the slowdown was jxl-oxide itself or thread contention,
a standalone benchmark compared the `image` crate integration path vs jxl-oxide's
direct `JxlImage` API, running single-threaded with 10 iterations:

```
--- JXL lossless 1080p ---
  image crate: 1920x1080 variant=Rgb16
  jxl-oxide direct: 1920x1080 channels=3
    breakdown: read=0.3ms parse=0.1ms render=70.2ms pixels=67.2ms
  image crate path:     min=100.0ms avg=109.8ms max=127.6ms
  jxl-oxide direct:     min=115.7ms avg=130.6ms max=162.9ms

--- JXL lossy 1080p ---
  image crate: 1920x1080 variant=Rgb16
  jxl-oxide direct: 1920x1080 channels=3
    breakdown: read=0.1ms parse=0.1ms render=51.1ms pixels=40.5ms
  image crate path:     min=95.7ms avg=106.1ms max=141.3ms
  jxl-oxide direct:     min=89.1ms avg=100.7ms max=135.3ms

--- PNG 1080p (baseline) ---
  PNG: 1920x1080 variant=other
  image crate path:     min=17.9ms avg=19.0ms max=23.6ms
```

### Findings

1. **Thread contention is the main amplifier.** Single-threaded: ~100ms. In-app with
   10 concurrent decode threads: ~280-300ms. The ~3x slowdown comes from threads
   fighting over CPU and the rayon thread pool (jxl-oxide uses rayon internally for
   parallel decoding).

2. **jxl-oxide is ~5x slower than PNG** for 1080p even in the best case (100ms vs 19ms).
   This is inherent to the format's complexity and the pure-Rust implementation.

3. **`write_to_buffer` (pixel extraction) takes ~40-67ms**, nearly as long as the
   render step itself. This is f32-to-u8 quantization and color management inside
   jxl-oxide.

4. **Rgb16 output variant.** jxl-oxide reports images as 16-bit (`variant=Rgb16`) even
   for 8-bit source content. This causes `image_to_color_image()` to hit the `other`
   fallback path which calls `into_rgba8()` for an extra 16-to-8-bit conversion.

5. **`fmt=None` from `with_guessed_format()`.** The `image` crate doesn't recognize
   JXL via its built-in format detection; it falls through to the hook-based decoder.
   Functionally correct but the `None` format may affect internal code paths.

### Third-party benchmark reference

From quackdoc's blog (5221x2847, ~14.8 MP test image on Ryzen 2600):
- libjxl (C++ reference): 200ms
- jxl-oxide: 562ms (~2.8x slower)

Extrapolated to 1080p (2 MP): ~45-80ms, consistent with our standalone ~100ms
(accounting for image content differences and non-linear scaling).

## Concurrency Fix (commit 2654d85)

Added `MAX_JXL_CONCURRENT = 2` cap in `cache.rs`. jxl-oxide uses rayon internally,
so 10 simultaneous JXL decodes all compete for the same CPU cores via rayon's
thread pool.

### Results

Per-decode latency improved:
- Before: ~280-340ms per 1080p JXL decode
- After: ~100-170ms per 1080p JXL decode

But skating FPS did not improve noticeably. The throughput is about the same:
- Before: 10 decodes at ~300ms = batch of 10 images every ~300ms
- After: 2 decodes at ~150ms = 2 images every ~150ms
- Both yield ~13-15 images/sec throughput

The fix makes delivery smoother (shorter stalls between images) but doesn't
increase total throughput, which is what gates skating FPS.

PNG decode is unaffected (still uses full thread count, ~35-43ms per decode).

### Remaining bottlenecks

1. **`variant=other` fallback**: jxl-oxide returns `Rgb16` for 8-bit content,
   forcing `image_to_color_image()` through `into_rgba8()` -- an extra conversion.
   Adding a direct Rgb16 fast path would save ~5-10ms per image.

2. **Inherent decode cost**: jxl-oxide's core render takes ~50-70ms for 1080p,
   pixel extraction another ~40-67ms. This is ~5x slower than PNG and likely
   irreducible without switching to libjxl (C bindings).

## Test Data

Converted 25 JXL test images at `/home/gota/ggando/rust_gui/data/demo/1080p_JXL_converted/`:
- 10 lossless (from 1080p PNGs, `-d 0 -e 7`): ~800-950K each
- 10 lossy (from 1080p PNGs, `-d 1.0 -e 7`): ~235-300K each
- 5 lossy (from small JPGs, `-j 0 -d 1.0 -e 7`): varied sizes

Conversion command: `cjxl input.png output.jxl -d 0 -e 7` (lossless)
