# FPS Baselines

Performance reference for development. Update when hardware, renderer, or caching changes land.

## Baseline 2026-09-16, `--bench-nav --bench-slider --bench-runs 3`, commit 10496f2

Linux (RTX 3090, 144 Hz), default settings (cache_count 5, decode_threads 10, lru 1024 MB, gpu Performance). Files in the OS file cache. Mean of 3 runs, min to max in brackets. Full reports: benchmarks/20260916_00*_gota-home_*.md, summary benchmarks/20260916_002541_gota-home_summary.md.

Skate (hold the arrow key), right pass / left pass:

| Folder | img/s right | img/s left | stall frames right | decode p50 |
|---|---|---|---|---|
| small_images, 4318 JPEG 800x800 | 144.7 (143.3 to 145.5) | 145.5 | 1% | 6.1 ms |
| 1080p_PNG_3MB, 101 PNG 1920x1080 | 131.1 (120.9 to 136.9) | 135.2 | 15% | 30.3 ms |
| 4k_PNG_10MB, 140 PNG 3840x2160 | 60.5 (58.1 to 62.9) | 59.0 | 64% | 76.1 ms |

Slider:

| Folder | phase | displayed % | UI block per load p50 | LRU hits | refill p50 | click-to-image p50 |
|---|---|---|---|---|---|---|
| small_images | sweep / scrub / jump | 45 / 48 / 100 | 5.0 ms | 15 / 131 / 0 | 31 / 39 / 35 ms | 14.1 ms |
| 1080p | sweep / scrub / jump | 100 / 100 / 100 | 23.4 ms | 79 / 105 / 0 | 105 / 118 / 132 ms | 65.6 ms |
| 4K | sweep / scrub / jump | 100 / 99 / 100 | 62.1 ms | 7 / 91 / 0 | 321 / 260 / 307 ms | 172.5 ms |

The 45 percent displayed on small images is the 10 ms slider throttle: decodes take 5 ms, frames 7 ms, so every second request is refused. The 1080p right pass has one low run (120.9, decode p50 33 ms) with nothing else running; the other two are 135 and 137.

## Current (v0.2.0-pre, wgpu + GPU LRU cache + mimalloc)

### macOS (M1 MacBook)

| Dataset | Key nav FPS | Slider FPS | Notes |
|---|---|---|---|
| 4K 10MB PNG | 50-55 | 14-16 | Occasional single stutter dropping to 30 in skate mode |
| 1080p 3MB | 60 | 40-45 | Consistent |
| small_images | 60 | 55-58 | |

### Linux (RTX 3090)

| Dataset | Key nav FPS | Slider FPS | Notes |
|---|---|---|---|
| 4K 10MB PNG | 60-64 | 14-15 | 60+ when lucky |
| 1K 3MB | 120+ | 30-40 | Reaches end quickly, uncertain ceiling |
| small_images | 145-146 | 61 | |

## Historical

### v0.1.0 era (devlog 027, decode_threads tuning)

| Platform | Dataset | Key nav FPS | Notes |
|---|---|---|---|
| Linux (RTX 3090) | 4K PNG | 60 | decode_threads=10, Balanced mode |
| Windows | 4K PNG | 50-54 | Performance mode, decode_threads=5 |
| macOS (M1) | 4K PNG | 50-56 | decode_threads=5 |
