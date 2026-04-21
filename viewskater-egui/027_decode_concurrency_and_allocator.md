# 027 — Decode concurrency, allocator retention, and mimalloc

Follow-up to 024–026. After moving the LRU to GPU (#2) and spreading
uploads across frames (#3), RSS on the 4K dataset was still 807 MB at
dir-open — barely changed from the original 872. Investigation found
the dominant cause isn't the GPU pipeline; it's Linux's default memory
allocator (glibc malloc) retaining pages after a transient burst of
parallel image decodes.

## The problem

`SlidingWindowCache::initialize` calls `spawn_load` for every non-center
slot. With `cache_count=5` that's 10 `std::thread::spawn` calls. Each
thread runs `image::open` → `image_to_color_image`, which transiently
allocates ~60 MB of CPU memory per 4K image (PNG decompression buffers +
RGB source + RGBA `Vec<Color32>` co-resident during the conversion).

10 threads in parallel → ~600 MB of simultaneous CPU heap usage. The
decoded `ColorImage`s are short-lived — they flow through the channel
into `pending_uploads`, get handed to `ctx.load_texture`, and drop
within a few frames. But glibc's malloc, once it grows its internal
arenas to hold the 600 MB peak, never returns those pages to the OS.

Evidence: after `close_images` (which drops all caches and calls
`malloc_trim(0)`), RSS stayed at 807 MB. The memory was freed from
Rust's perspective, but glibc's arena pages remained mapped.

## How iced avoids this

Explored the iced viewskater codebase (`~/ggando/rust_gui/viewskater/`)
to understand why it reports 313 MB on the same data.

Key difference: iced's `ImageCache` (`src/image_cache.rs:76`) stores
raw encoded file bytes (`Vec<Option<Vec<u8>>>`), not decoded pixels.
Its loader (`file_io.rs::async_load_image`) just reads the `.png` file
into a buffer (~10 MB per 4K PNG). 10 parallel file reads = ~100 MB
peak, not 600 MB.

Actual decode (the expensive RGB→RGBA conversion) happens lazily inside
iced's renderer when `Handle::from_bytes` is drawn — one image at a
time, serialized by the render pipeline. No parallel decode burst ever
occurs.

## Experiment: serializing decodes (MAX_CONCURRENT_DECODES=1)

Added `MAX_CONCURRENT_DECODES` to cap how many background decode threads
run simultaneously. With a cap of 1:

| Metric | Before (unlimited) | MAX_CONCURRENT=1 |
|---|---|---|
| RSS after 4K dir open | 807 MB | **308 MB** |
| RSS after full scrub | 862 MB | **363 MB** |
| Keyboard nav FPS | 60 | **14** |

308 MB matches iced's 313 MB almost exactly. This confirms the parallel
decode burst is the entire 500 MB gap.

FPS dropped to 14 because each 4K PNG decode takes ~60 ms. With one
decode at a time, the cache fills at ~16 images/sec. When the user
holds the arrow key, they outrun the 5-image cache buffer within 83 ms
and hit the decode bottleneck.

MAX_CONCURRENT=4 gave 50-55 FPS and presumably ~400-500 MB RSS —
better but still a compromise. There's no value of MAX_CONCURRENT that
gives both 60 FPS and low RSS, because the problem isn't how many
decodes run — it's that glibc retains the peak memory afterward.

## Fix: replace glibc malloc with mimalloc

mimalloc (Microsoft's allocator, used in .NET, Edge, etc.) actively
returns memory to the OS after transient spikes via `madvise(DONTNEED)`
on freed pages. Unlike glibc, it doesn't permanently grow its arena
pool.

Change: 2 lines in `Cargo.toml` + 3 lines in `main.rs`:

```rust
#[global_allocator]
static GLOBAL: mimalloc::MiMalloc = mimalloc::MiMalloc;
```

This replaces the allocator for the entire process. All 10 concurrent
decodes still run (preserving 60 FPS). After the transient burst,
mimalloc should release the pages and RSS should settle near the
steady-state footprint.

Also set `MAX_CONCURRENT_DECODES = 10` (matching old behavior) so FPS
is not affected. The decode-throttle infrastructure (`pending_decodes`
queue, `spawn_thread` split) stays in place for future tuning if needed.

## Decode-throttle infrastructure (kept but not limiting)

Regardless of the allocator fix, `spawn_load` was split into:

- `spawn_load`: checks `in_flight.len() < MAX_CONCURRENT_DECODES`.
  If at capacity, pushes to `pending_decodes` queue instead of spawning.
- `spawn_thread`: the actual `std::thread::spawn`.
- `poll()`: when a decode result arrives and a slot opens, pops the
  next entry from `pending_decodes` and spawns it.

With MAX_CONCURRENT=10 this is effectively unlimited for an 11-slot
window, matching old behavior. If we ever need to tune concurrency
(e.g., on low-memory devices), the knob is there.

## Result

mimalloc confirmed the hypothesis and fixed the retention. On the 4K
PNG dataset, Linux, RTX 3090, Balanced GPU memory mode:

| Moment | glibc (before) | mimalloc (after) |
|---|---|---|
| Launch | 251 MB | ~250 MB |
| Dir-open peak (transient) | 872 MB (stays) | 880 MB (transient) |
| Dir-open settled | **807 MB** | **345 MB** |
| Slider scrub | 862 MB | ~400 MB |
| Slider release peak | — | 1000 MB (transient) |
| Slider release settled | — | **373 MB** |

The transient spikes still happen (10 concurrent decodes still peak at
~600 MB of CPU memory) but mimalloc returns the pages within a few
hundred milliseconds. glibc held them permanently.

Steady-state 345–373 MB vs iced's 313 MB. Within ~30–60 MB, likely
accounted for by egui's TextureManager overhead and structural
differences between the frameworks.

Keyboard nav FPS preserved at 60 (MAX_CONCURRENT_DECODES=10 kept).

## Caveats

mimalloc is MIT-licensed, maintained by Microsoft Research. Production
use in .NET runtime, Zig's standard library, various game engines.

- Adds a C library build dependency (libmimalloc-sys compiles ~30
  C files). Build time increase: ~2 seconds.
- Binary size increase: ~100-200 KB.
- Cross-platform: works on Linux, macOS, Windows, FreeBSD. No
  platform-specific code paths needed.
- `default-features = false` disables the `secure` mode (guard pages
  on every allocation — slower, not needed for an image viewer).

Alternative allocators considered: jemalloc (used in Firefox, Redis) —
similar behavior, heavier build, Linux-focused. mimalloc is lighter
and more portable.
