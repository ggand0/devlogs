# 024 — GPU memory counter in FPS overlay

Adds a real GPU memory readout to the `Img: ...` overlay so we can measure
the upcoming GPU-backed LRU refactor (025) without needing `nvidia-smi` in
another window. Splits CPU RSS from wgpu-hal's tracked GPU bytes in the
display.

## What ships

- New display format: `Img: 14.8 FPS | RSS:628 GPU:353 (L:180 C:348)`
  - `RSS:` — process RSS from `sysinfo` (was shown before as the unlabelled
    `872 MB` number).
  - `GPU:` — sum of wgpu-hal's `buffer_memory + texture_memory +
    acceleration_structure_memory` counters, in MB.
  - `(L C)` — logical LRU bytes and sliding-window bytes. Same as before.
    Left in place; it's our own bookkeeping and still useful as a
    sanity-check against `GPU:`.
- `Cargo.toml`: direct dep on `wgpu = { version = "24", features =
  ["counters"] }`. Cargo feature unification turns the `counters` feature on
  in the single shared wgpu crate, so eframe's `egui-wgpu` also sees the
  instrumented build.
- `src/perf.rs::ImagePerfTracker::sample_gpu_memory(&wgpu::Device)` — 1 Hz
  throttled sampler, stores bytes on the tracker. Also emits a
  `log::debug!` breakdown line per sample so we can read per-resource
  detail (tex/buf/accel + texture count + allocation count) without adding
  more UI.
- `src/app.rs::update()` — calls `perf.sample_gpu_memory(&device)` right
  after the existing `device.poll(Wait)`. Same gate: only if
  `frame.wgpu_render_state().is_some()`.

## Why wgpu's `counters` feature matters

`wgpu_types::InternalCounter` is an `AtomicIsize` gated on
`#[cfg(feature = "counters")]`. Without the feature, every `.read()` call
returns a hard-coded `0`. So reading `device.get_internal_counters()` on a
default wgpu build compiles and runs fine — it just always gives zero.

The counters feature chains: `wgpu/counters` → `wgpu-core/counters` →
`wgpu-types/counters`. Eframe's `egui-wgpu` depends on wgpu with
`default-features = false`, so the feature is off by default. Adding wgpu
directly with `features = ["counters"]` gets cargo's feature unification to
merge it into the single wgpu that eframe also uses.

Verified it was actually active by reading
`target/opt-dev/.fingerprint/wgpu-types-*/lib-wgpu_types.json` —
`"features":"[\"counters\", \"default\", \"std\"]"`.

## Verification

On the 4K PNG dataset (`data/demo/4k_PNG_10MB`, 3840×2160 RGBA,
`cache_count=5` → 11-slot sliding window):

```
idle (before open):        textures=1  tex=1MB   buffers=5 allocations=2
after loading directory:   textures=12 tex=353MB buffers=5 allocations=7
```

The arithmetic lines up:

- 1 × egui font atlas + 11 × sliding-window textures = **12 textures**.
- 3840 × 2160 × 4 = 33,177,600 B ≈ 31.64 MiB per texture.
- 11 × 31.64 = **348 MiB** of image data + a few MB font atlas ≈ **353 MB**.
- `allocations=7` (vs 12 textures) shows gpu_allocator's block-pooling:
  multiple textures share each 64/128 MB Manual-mode block (the Balanced
  GPU memory hint from #13b3484). Dedicated allocations would have given
  `allocations=12`.

The `C:` column (`348`) is our own sum of `width × height × 4` over
sliding-window slots, computed independently of wgpu. It matches the
counter's `tex` value to within ~1 MB. Two independent paths agreeing is
the check that the counter is reading real allocations, not garbage.

## What this unblocks

Now we can measure the 025 CPU→GPU LRU refactor directly:

- Baseline now: `GPU:353` with the LRU on CPU (the LRU's 180 MB shows up in
  RSS, not GPU).
- After 025: `GPU:` should rise by whatever portion of the old CPU LRU
  ends up held as additional GPU textures, and RSS should drop by the same
  amount (minus glibc pool hysteresis). The handoff doc's prediction was
  RSS 872 → ~430–460 MB on the 4K dataset.

Without this counter the refactor would be flying blind: RSS alone can't
distinguish "pixels moved to GPU" from "pixels leaked."

## Gotchas

- On discrete GPUs the wgpu counter is **not** in RSS. It's device-local
  VRAM. So `RSS + GPU` is not a meaningful total; they live in different
  address spaces. On integrated GPUs they overlap. The overlay shows them
  side by side without summing, which is correct.
- `memory_allocations` is the count of gpu_allocator blocks, not the
  texture count. Don't confuse it with `textures`.
- Sampling is 1 Hz. Matches the CPU RSS throttle. Debug log line fires at
  the same cadence.
