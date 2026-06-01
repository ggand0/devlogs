# FPS Baselines

Performance reference for development. Update when hardware, renderer, or caching changes land.

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
