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

## Re-verification and Future Work (2026-09-07)

Written by: Claude Fable 5.1 (claude-fable-5-1), with the owner.

Notes from picking the branch back up after a break, before the rebase onto
main. The branch is exactly upstream 576e928 plus uncommitted timing logs.

### Effect on other formats

The PR cannot change JPG/PNG decode performance. What
`jxl_oxide::integration::register_image_decoding_hook()` installs in the
`image` crate (0.25.9), from reading the crate sources:

- Two format-detection hooks (`GUESS_FORMAT_HOOKS`), one per JXL magic
  sequence (codestream and container). `ImageReader::guess_format` checks
  these before the built-in `MAGIC_BYTES` table on every open. Cost: a
  `starts_with` on the same 16-byte header the built-in guess already reads.
- One decoding hook (`DECODING_HOOKS`) keyed by the extension string `"jxl"`.
  `make_decoder` only consults the map when the format resolved to
  `Format::Extension`, i.e. when a detection hook matched. Built-in formats
  take the `Format::BuiltIn` arm and never touch the map.
- jxl-oxide's `JxlDecoder::new` uses `JxlThreadPool::rayon_global()`, so JXL
  decodes run on rayon's global pool, which rayon initializes on first use.
  `rayon` was already in the dependency graph before this PR; no pool exists
  until the first JXL decode, and PNG/JPG decoders do not use it.

The mixed PNG/JXL skate runs above measured this directly: PNG decode p50
was 28-30ms with JXL decodes saturating the rayon pool alongside, the same
as in a PNG-only folder. The only global cost of the PR is the added crates
(jxl-* family, brotli-decompressor) in compile time and binary size.

### Test directories under `data/demo/`

| Directory | Contents | Purpose |
|---|---|---|
| `1080p_JXL_converted` | 25 files: 10 lossless, 10 lossy, 5 from small JPGs | first-round latency benches |
| `1080p_JXL_lossless` | 101 files, the full 1080p PNG set at `-d 0 -e 7` | uniform folder for manual skating |
| `skate_test_jxl_400` | 400 real copies cycling the lossless frames | 12s skate throughput runs |
| `skate_test_mixed_900` | 900 files alternating PNG / JXL | mixed-folder regression check |

The 400/900 sets exist only so a 12s skate cannot exhaust the folder.

### Why skating stalls every ~5 images

Manual skating through `1080p_JXL_lossless` shows a burst-and-stall pattern:
about 5 images advance quickly, then a visible pause, at 17-20 fps average.
Single A/D presses feel smoother. The mechanism is the prefetch depth vs
decode latency:

- `cache_count` defaults to 5, so the window holds 5 images ahead.
- Skating advances only when the next slot already has a texture
  (`Pane::is_next_cached`), at the key-repeat rate of ~30Hz.
- Each advance spawns one decode 5 slots ahead
  (`ImageCache::navigate_forward`), which completes 150-200ms later at full
  concurrency.

So 5 prefetched images are consumed in ~170ms, the cursor reaches the edge
of the pipeline, and waits for the next batch. Single presses are slower
than 30Hz, so the pipeline stays ahead of the cursor. Either way the average
is the decode throughput, ~22 img/s for 1080p lossless on the 3900X.

Raising `cache_count` does not help: it lengthens bursts and stalls equally,
because 30Hz consumption always outruns 22 img/s.

### Candidate fixes for a follow-up PR

In order of expected payoff:

1. **Pace skating to decode throughput.** Instead of advancing at 30Hz until
   the pipeline runs dry, advance at the measured decode completion rate.
   Same average, but steady instead of burst-and-stall. Cheap, and helps
   every slow format (large TIFF, JP2), not just JXL.
2. **Benchmark realistic files.** Lossless-from-PNG is the worst case for
   jxl-oxide (modular mode), and libjxl only gained 15-30% there. Real-world
   JXL is mostly lossy VarDCT or losslessly transcoded JPEG (`cjxl -j 0`),
   which decodes faster. Measure that before optimizing further.
3. **Rgb16 fast path in `image_to_color_image`.** jxl-oxide reports 16-bit
   even for 8-bit sources, so every JXL takes the generic `into_rgba8()`
   fallback. Worth ~5-10ms per 1080p image.
4. **u8 output via the direct jxl-oxide API.** Pixel extraction
   (`write_to_buffer`) was 40-67ms of the ~100ms, and the earlier direct-API
   bench compared f32 paths only. If jxl-oxide can render to u8 and skip
   color management for sRGB sources, this may be the largest single win.
   Unverified; needs a bench.
5. **Reduced-resolution decode while skating.** JXL carries a 1/8 LF image.
   Decoding only that during skating and the full image on pause would
   break the throughput ceiling entirely. Not checked whether jxl-oxide
   exposes it; research item.

### Measured: where JXL decode time goes (2026-09-07)

Follow-up measurements to turn the candidate list above into numbers. All
on the 3900X (12c/24t), opt builds, 1080p. Scratch benches lived in the
session scratchpad (`jxlstage/`: stage breakdown, `jxltp`: throughput with
selectable jxl-oxide thread pool, `pxconv`/`planar`: extraction paths).

**Every earlier benchmark used 16-bit data.** `jxlinfo` on the test sets
reports "16-bit RGB" for both the lossless and lossy sets, because the
source `1080p_PNG_3MB` PNGs are 16-bit/color. jxl-oxide's `Rgb16` output
was correct, not a quirk, and 16-bit lossless is its worst case. New 8-bit
sets were built from frames 1-10 of `1080p_PNG_3MB` and saved under
`data/demo/1080p_8bit_sets/` (10 files each):

```
png8/      convert -depth 8                       8-bit PNG source
jpg/       convert png8 -quality 92 -> JPEG       JPEG source
ll8/       cjxl png8 -d 0 -e 7                    8-bit lossless
lossy8/    cjxl png8 -d 1 -e 7                    8-bit lossy
jpgt/      cjxl jpg (default: lossless transcode) JPEG recompressed
jpglossy/  cjxl jpg -j 0 -d 1 -e 7                JPEG re-encoded lossy
```

Throughput via the app's decode path (`bench_jxl_throughput`, image crate +
hook + rgba8), concurrency 1 / 10:

```
16-bit lossless (old set)   conc=1  9.5 img/s   conc=10 16-24 img/s
16-bit lossy (old set)      conc=1 10.6 img/s   conc=10 31.5 img/s
8-bit lossless              conc=1 10.3 img/s   conc=10 24.0 img/s
8-bit lossy d1              conc=1  8.1 img/s   conc=10 28.0 img/s
JPEG q90 transcode          conc=1  9.4 img/s   conc=10 39.5 img/s
JPEG -> lossy d1            conc=1  9.7 img/s   conc=10 32.8 img/s
PNG 8-bit                   conc=1 49.4 img/s   conc=10 321 img/s
JPEG q90                    conc=1 70.9 img/s   conc=10 423 img/s
```

Stage breakdown, direct jxl-oxide API, one decode at a time
(read / parse are <0.5ms and omitted):

```
                         rayon=24 threads          rayon=1 thread (CPU cost)
                         render   px_u8   total    render   px_u8   total
8-bit lossless            77ms    51ms    128ms    455ms    46ms    502ms
8-bit lossy d1            40ms    52ms     92ms    100ms    43ms    143ms
JPEG transcode            40ms    42ms     82ms     73ms    42ms    115ms
16-bit lossless (old)     77ms    55ms    132ms    438ms    54ms    492ms
```

Findings:

1. **Pixel extraction is a flat 42-55ms per 1080p image on any thread
   count, format, or bit depth.** `ImageStream::write_to_buffer` is a
   per-sample loop with coordinate transforms and grid lookups
   (`jxl-oxide/src/fb.rs`). `Render::image_all_channels()` costs 25-36ms
   for the same reason; a tight f32->u8 loop over its output is 5ms.
   `image_planar()` + interleave is 49-54ms. The raw `ImageWithRegion` is
   private on `Render`, so the app cannot bypass this; the fix is upstream
   in jxl-oxide (row-wise copy from the grids, then convert). That is ~30%
   of the CPU cost of a lossy decode and ~40% of a JPEG-transcode decode.
2. **Render parallelism inside one image is poor.** 24 rayon threads give
   2.5x over 1 thread for lossy and 6x for lossless. Cross-image
   parallelism (our decode threads) is what delivers throughput.
3. **Throughput plateaus at ~33-37 img/s (lossy) from 8 concurrent decodes
   up**, whichever pool jxl-oxide uses: global rayon, a per-decode
   single-threaded pool (`JxlThreadPool::none()`), or per-decode 2/4-thread
   pools. CPU per image rises from 173ms at conc=1 to ~230ms at conc=8 and
   only ~7.6 cores stay busy out of 24, so the pipeline is memory-bound
   (f32 planar buffers, ~25MB per pass per image), not CPU-bound. glibc
   page faults were 6s of 22s CPU in the bench; the app uses mimalloc, so
   that part is already mitigated there.
4. **4-core simulation** (`taskset -c 0-3`, and with SMT siblings):

```
                 4 threads                      8 threads (4c + SMT)
                 global c=10   none c=4         global c=10   none c=8
8-bit lossy      20.0 img/s    20.4 img/s       23.1 img/s    27.2 img/s
JPEG transcode   22.5 img/s    25.7 img/s       31.4 img/s    29.5 img/s
8-bit lossless    7.8 img/s     7.7 img/s       10.5 img/s    10.2 img/s
avg latency      420-1280ms    150-500ms        300-950ms     250-760ms
```

   Same throughput either way; a per-decode single-threaded pool halves
   latency at laptop core counts because decodes stop queueing on one
   shared pool. A single decode alone (opening one file) is 2.5x faster on
   the global pool, so the right split is global pool for the current
   image, single-threaded pool for prefetch decodes.
5. **Reduced-resolution decode is not available.** jxl-oxide 0.12 exposes
   no downsampled or LF-only render; the LF frame is internal to
   `jxl-render`. Drop that candidate unless upstream adds it.

Revised candidate list, by payoff:

1. Upstream a fast extraction path in jxl-oxide (or carry a patched fork via
   `[patch.crates-io]`): ~30-40% more JXL throughput everywhere.
2. Per-decode `JxlThreadPool::none()` for prefetch decodes via a custom
   `image` hook (`JxlDecoder::with_thread_pool`), global pool for the
   current image: lower latency on laptops, no throughput cost.
3. Pace skating to decode completion rate (UX, all slow formats).
4. libjxl via `jpegxl-rs`: ~2x for lossy only, C build dependency. Still
   not recommended.

Realistic expectation: 8-bit lossy and JPEG-transcoded JXL, which is what
exists in the wild, skates at 30-45 img/s on this desktop and ~20-25 img/s
on a 4-core laptop. 16-bit or lossless-from-PNG JXL stays at 8-10 img/s on
a laptop no matter what the app does.

### Test set: `data/demo/jxl_wild/` (2026-09-07)

The old JXL sets are 16-bit lossless/lossy conversions of 16-bit PNGs,
jxl-oxide's worst case. `jxl_wild/` is a flat 100-file, 81MB folder built to
look like a JXL folder someone would actually have. All 8-bit, all `cjxl`
0.11.1 at the default effort 7:

```
40  jpegxl_transcode_4k_NN.jxl   3840x2160  4k_PNG_10MB -> JPEG q92 -> `cjxl in.jpg`
                                             (lossless JPEG recompression, the
                                             default cjxl behavior for JPEG
                                             input; how most existing JXL
                                             files are made)
30  lossy_d1_4k_NN.jxl           3840x2160  4k_PNG_10MB -> `cjxl -d 1 -e 7`
                                             (cjxl default for non-JPEG input;
                                             camera / editor export)
10  lossy_d1_1080p_NN.jxl        1920x1080  1080p_PNG_3MB -> 8-bit -> `cjxl -d 1 -e 7`
20  lossless_1080p_NN.jxl        1920x1080  1080p_PNG_3MB -> 8-bit -> `cjxl -d 0 -e 7`
                                             (screenshots, art)
```

4K is the majority because wild JXL is mostly photos, and photos are not
1080p. Sources: 4K PNGs 1-70 and 1080p PNGs 11-40, so no overlap with the
frames used in the older sets.

Throughput through the app decode path, 10 threads, 3900X:

```
jpegxl_transcode_4k    10.0 img/s   avg latency 1001ms
lossy_d1_4k             7.8 img/s   avg latency 1280ms
lossy_d1_1080p         31.3 img/s   avg latency  320ms
lossless_1080p         23.8 img/s   avg latency  421ms
whole folder           10.8 img/s   avg latency  924ms
```

Stage breakdown for one 4K image (transcode / lossy):

```
                       rayon=24                rayon=1
                       render  px_u8  total    render  px_u8  total
JPEG transcode 4K      196ms   171ms  368ms    283ms   177ms  461ms
lossy d1 4K            185ms   187ms  372ms    478ms   183ms  661ms
```

At 4K the pixel-extraction stage is as long as the render itself, and it
does not parallelize. The realistic wild folder skates at ~10 img/s on this
desktop, which on a 4-core laptop extrapolates to ~4-5 img/s. Fixing
extraction upstream (candidate 1 above) is worth close to 2x at 4K, more
than any other option.

In-app check on `jxl_wild` (manual skate, default settings): same
burst-of-5-then-stall pattern as the lossless set, 8-10 fps in the 4K
files and 10-15 fps in the 1080p files. 4K matches the isolated bench
(7.8-10 img/s). 1080p reads lower than its isolated bench (24-31 img/s)
because the folder sorts by name, so the 30 1080p files sit between 4K
groups: entering them, all 10 decode threads are still busy with ~1s 4K
decodes from the trailing window, and 30 files is too short a run to reach
steady state.

Same folder on a MacBook Pro M1: similar fps to the 3900X. Consistent with
the pipeline being memory-bound rather than core-bound (M1 has a third of
the threads but high memory bandwidth and strong single-core).

### Measured: PNG decode latency next to JXL decodes, by core count (2026-09-07)

The July mixed-folder result (PNG p50 unchanged) was 24 threads only. Same
question at laptop core counts, via `taskset`. App-style path (image crate +
jxl hook + rgba8), 10 decode threads, files alternating PNG / lossy JXL.
Data: 1080p = `1080p_8bit_sets/png8` + `1080p_8bit_sets/lossy8` (10 + 10);
4K = first 10 of `4k_PNG_10MB` + first 10 of `jxl_wild/lossy_d1_4k_*`.
100 decodes per 1080p run, 40 per 4K run, half PNG half JXL. PNG-only runs
use the same PNGs alone.

```
                          PNG only          PNG + JXL alternating
                          png p50 / p90     png p50 / p90     jxl p50
4 threads   1080p         48 / 92 ms        29 / 55 ms        440 ms
            4K            147 / 388 ms      160 / 218 ms      1569 ms
8 threads   1080p         33 / 65 ms        32 / 51 ms        385 ms
            4K            136 / 219 ms      124 / 200 ms      1457 ms
24 threads  1080p         27 / 39 ms        35 / 50 ms        287 ms
            4K            145 / 201 ms      178 / 269 ms      1136 ms
```

PNG decode latency does not get worse next to JXL decodes at 4 or 8
threads. It even drops at 4 threads because half of the 10 decode threads
are parked waiting on rayon instead of competing for the 4 cores. At 24
threads it rises ~20%, consistent with the memory-bandwidth limit seen
earlier. The JXL decodes themselves are the bottleneck at every core count
(440 ms per 1080p, 1.5 s per 4K on 4 threads).
