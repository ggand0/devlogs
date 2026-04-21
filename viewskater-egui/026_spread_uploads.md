# 026 — Spread SlidingWindowCache uploads across frames

Item #3 from the feat/gpu-cache plan. Changes `SlidingWindowCache::poll`
to issue at most `UPLOADS_PER_FRAME = 2` GPU uploads per frame, buffering
completed decodes in a `pending_uploads` queue. Ships to avoid the
11-uploads-in-one-frame spike on directory open.

## What ships

- `SlidingWindowCache.pending_uploads: VecDeque<(usize, ColorImage, String)>` —
  completed decodes waiting to be handed to `ctx.load_texture`.
- `poll()` now two-phase: drain `rx` into `pending_uploads`, then issue
  up to 2 uploads. If the queue is non-empty after, call
  `ctx.request_repaint()` so the next frame runs even without user input.
- `initialize()` clears `pending_uploads` so stale decodes from a previous
  window don't get uploaded into unrelated slots.
- Upload step guards against stale slots (window shifted since decode)
  and against double-fill (some other decode already populated the slot).

## Measured result

On 4K PNG dataset, Linux, Balanced GPU memory mode, RTX 3090:

|                        | Before #3 | After #3 | Delta |
|------------------------|-----------|----------|-------|
| After opening 4K dir   | 872 MB    | 807 MB   | -65 MB |
| After full slider scrub| 928 MB    | 862 MB   | -66 MB |

Much smaller than the ~300 MB the handoff doc's theory predicted. #3's
architectural improvement is real (staging-belt peak size is bounded,
retention can't exceed the bounded peak) but it doesn't move the
headline number because **the dominant CPU peak on directory open is
not the staging pool**.

## Why the saving is only 65 MB — decode concurrency is the real bottleneck

`close_images` calls `malloc_trim(0)` but RSS stays at 807 MB after
dropping all textures. That tells us the 556 MB above launch baseline is
not reclaimable via glibc — it's either in wgpu/gpu_allocator's host
pools, in the NVIDIA driver, or — the likely culprit — in glibc arenas
that `malloc_trim` can't unmap because the pages are fragmented.

The real suspect: `SlidingWindowCache::spawn_load` spawns `std::thread::spawn`
for every non-center slot in `initialize()`. On open, 10 threads start
decoding in parallel. Each 4K decode allocates:

- PNG decompression buffers (decoder-internal)
- `DynamicImage::ImageRgb8` buffer: ~24 MB
- `Vec<Color32>` result: ~33 MB (co-resident with source until `collect`
  finishes consuming it)

Peak ≈ 50–60 MB per in-flight decode. With 10 concurrent: **500–600 MB
of decode arena pressure at dir open**. Once glibc expands arenas to
hold that, it retains the pages.

iced avoids this by using sequential async tasks — exactly one decode in
flight at a time. Its reported steady-state RSS on the same data is
~313 MB after open, ~520 MB after slider scrubbing. The 500 MB gap
vanishes if we serialize decodes.

## Not fixed here

Decode-concurrency throttling. That's the actual RSS lever. Proposed
for a follow-up: single dedicated decoder thread consuming a
`channel::<DecodeRequest>`, or a semaphore-bounded thread pool of ≤2
decoders. Mirrors iced's one-at-a-time load pattern.
