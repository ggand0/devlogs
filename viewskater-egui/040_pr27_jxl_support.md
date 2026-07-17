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

## libjxl (C) Benchmark Comparison

Benchmarked `jpegxl-rs` (Rust bindings to libjxl C++ reference decoder) with
multithreaded `ThreadsRunner`, against jxl-oxide via both the `image` crate
integration and the direct API. 10 iterations, single-threaded, opt-dev profile.

```
--- JXL lossless 1080p ---
  image+jxl-oxide:      min=99.3ms  avg=103.5ms max=108.4ms
  jxl-oxide direct:     min=112.6ms avg=126.7ms max=158.8ms
  libjxl (C, threaded): min=75.4ms  avg=88.3ms  max=119.2ms

--- JXL lossy 1080p ---
  image+jxl-oxide:      min=95.4ms  avg=106.6ms max=123.2ms
  jxl-oxide direct:     min=94.4ms  avg=104.3ms max=121.3ms
  libjxl (C, threaded): min=44.1ms  avg=50.2ms  max=61.3ms

--- PNG 1080p (baseline) ---
  image crate:          min=18.2ms  avg=21.7ms  max=46.5ms
```

libjxl is ~2x faster for lossy (50ms vs 104ms) and ~15-30% faster for lossless
(88ms vs 104ms). Not a dramatic enough improvement to justify adding a C build
dependency to a pure-Rust project. The `jpegxl-rs` dev-dependency was removed
after benchmarking.

## Conclusion

JXL decode performance at 1080p (~100ms per image with jxl-oxide) is inherently
~5x slower than PNG (~20ms). This caps skating FPS at ~15. No optimization on
our side can close this gap without replacing jxl-oxide, and libjxl (C) only
offers a ~2x improvement which doesn't justify the build complexity tradeoff.

The PR is correct and the concurrency fix reduces per-decode latency. JXL images
load and render correctly; skating is slower than PNG but functional.

## Test Data

Converted 25 JXL test images at `/home/gota/ggando/rust_gui/data/demo/1080p_JXL_converted/`:
- 10 lossless (from 1080p PNGs, `-d 0 -e 7`): ~800-950K each
- 10 lossy (from 1080p PNGs, `-d 1.0 -e 7`): ~235-300K each
- 5 lossy (from small JPGs, `-j 0 -d 1.0 -e 7`): varied sizes

Conversion command: `cjxl input.png output.jxl -d 0 -e 7` (lossless)

## Follow-up Review: Auditing the Concurrency Cap (2026-07-16)

A second review pass before the merge decision, re-examining commit 2654d85
(`MAX_JXL_CONCURRENT = 2`) with fresh benchmarks. The result contradicts the
earlier conclusion: the cap does reduce throughput, and the "both yield ~13-15
images/sec" claim above was wrong.

### Method

Two benchmarks, both on the Ryzen 9 3900X (12c/24t):

1. **Standalone throughput bench** (`examples/bench_jxl_throughput.rs`):
   replicates the app's decode path from `cache.rs::spawn_thread` (one std
   thread per decode, `open_image()` + rgba8 conversion), pulling from a
   shared queue of the 20 1080p test JXLs. Sweeps worker-thread count and
   measures aggregate images/sec.

2. **In-app skate simulation**: built a 400-image directory of symlinks
   cycling through the 20 test JXLs, launched the real app with
   `RUST_LOG=viewskater=debug`, held Shift+Right for 12s via
   `xdotool keydown`, and computed decode completions per second from the
   `bg decode` log lines. In steady-state skating the advance rate equals the
   decode completion rate, so this measures skating FPS. The cap was made
   env-overridable via a temporary patch (reverted after benching).

### Standalone results (JXL-only)

```
concurrency= 1  throughput=10.1 img/s  avg_latency=99ms
concurrency= 2  throughput=15.9 img/s  avg_latency=126ms
concurrency= 4  throughput=20.8 img/s  avg_latency=193ms
concurrency= 8  throughput=23.7 img/s  avg_latency=337ms
concurrency=12  throughput=23.6 img/s  avg_latency=509ms
```

Restricting jxl-oxide's internal rayon pool instead (parallelize across
images, not within) collapses throughput: `RAYON_NUM_THREADS=1` yields
3.8 img/s at any concurrency because all decodes serialize through the shared
global pool. The default pool size is correct.

### In-app skate results (JXL-only, 400 images)

```
cap=2 (current)  ~15 img/s   decode p50=121ms p90=150ms max=222ms
cap=4            ~20 img/s   decode p50=179ms p90=244ms max=296ms
cap=6            ~20 img/s   decode p50=226ms p90=287ms max=399ms
cap=8            ~21 img/s   decode p50=208ms p90=276ms max=404ms
cap=10 (no cap)  ~20 img/s   decode p50=214ms p90=298ms max=546ms
```

Totals cross-check: 12s x 20/s + ~11 initial window fills = 251, observed
255-269 per run. The cap=2 run: 12 x 15 + 11 = 191, observed 200.

Uncapped skating is ~20 img/s, not 13-15. The earlier "throughput is about
the same" conclusion came from comparing per-decode latencies and inferring
throughput, not from measuring it. The ~300ms per-decode latency at 10
in-flight decodes is queueing on the shared rayon pool, not wasted work:
the pool stays saturated either way, so total throughput is unharmed.

### Mixed PNG/JXL directory

The cap logic special-cases JXL in the shared `pending_decodes` queue, so a
mixed directory is the interesting case for regressions. Test: 400 symlinks
alternating 1080p PNG / 1080p JXL, same 12s skate.

```
                 advance rate   PNG decode p50   JXL decode p50
cap=2            ~28 img/s      30.1ms           128ms
cap=10 (no cap)  ~35 img/s      28.8ms           143ms
```

The uncapped run swept all 400 images before the 12s hold ended, so its
steady-state rate (35-39/s mid-run) is a lower bound. Two findings:

- PNG decode latency is unchanged with or without the cap (~29-30ms p50).
  The cap protects PNGs from nothing measurable; jxl-oxide's rayon work does
  not meaningfully starve PNG decode threads at this image size.
- Uncapped mixed skating is ~20-25% faster because JXL delivery, which gates
  the advance rate, is no longer throttled.

### Code audit of the cap logic

Two latent defects found in the `MAX_JXL_CONCURRENT` implementation, both
consequences of bolting a per-format limit onto the shared FIFO queue:

1. **Head-of-line blocking** (`cache.rs` poll refill loop): when the front of
   `pending_decodes` is a JXL at the cap, the loop `break`s instead of
   scanning past it, stalling any queued PNGs behind it even with free
   global slots. Rarely triggered at the default `decode_threads = 10`
   (PNGs only queue when 8+ non-JXL decodes are in flight), but plausible at
   low `decode_threads` settings (the slider allows 1-16).

2. **Stale-result counter skew**: `initialize()` resets `jxl_in_flight = 0`
   while previously spawned threads may still be running. When such a stale
   decode completes, `poll()` decrements the counter for a decode that was
   never counted in the new epoch. `saturating_sub` prevents underflow, but
   if new-epoch JXLs are in flight the decrement lands on them, letting the
   effective cap drift above 2 until the next `initialize()`.

### Revised conclusion

The contention diagnosis behind commit 2654d85 was mistaken: the latency
inflation at high concurrency is benign queueing, and capping concurrency at
2 costs ~25% skating throughput in JXL-only directories (15 vs 20 img/s) and
~20-25% in mixed directories (28 vs 35+ img/s) while leaving PNG latency
unchanged. The cap's only benefit is a smaller latency tail (nearest
neighbors fill slightly sooner after a jump), which does not justify the
added state, the head-of-line blocking hazard, or the counter skew.

Recommendation: drop the concurrency cap (revert 2654d85) and merge the
upstream PR as-is. The realistic skating ceiling for 1080p JXL on this
machine is ~20 img/s vs 60+ for PNG, inherent to jxl-oxide's decode cost.

### Verification after dropping the cap

Commit 2654d85 was dropped from the branch (backed up as a format-patch in
`tmp/jxl_pr27_backup/`, restorable via `git am`). Re-ran the same 12s skate
tests on the new HEAD (576e928, upstream PR only), with the mixed directory
enlarged to 900 images so it could not be exhausted mid-run:

```
                    advance rate   vs cap=2 baseline
JXL-only (400)      ~22 img/s      ~15 img/s  (+45%)
mixed PNG/JXL (900) ~34 img/s      ~28 img/s  (+20%)
```

Mixed-run PNG decode latency p50=29.1ms (unchanged), JXL p50=145.7ms.
Totals cross-check: JXL-only 277 decodes = 12s x 22 + 11 window fills;
mixed 411 = 12s x 33 + 11. Matches the cap=10 predictions from the sweep.

Repeated with real file copies instead of symlinks
(`data/demo/skate_test_jxl_400/`, 219MB, and `data/demo/skate_test_mixed_900/`,
1.5GB, built by cycling the 20 test JXLs and 101 test PNGs) to rule out any
symlink artifact: JXL-only ~22 img/s (282 decodes, latency p50=192ms), mixed
~33 img/s (407 decodes, PNG p50=28.1ms, JXL p50=148.1ms). Identical to the
symlink runs within noise.
