# 022 iced version memory comparison: where the gap actually lives

Branch: `feat/renderer-selection` (background investigation, not implemented)

## Summary

After tuning wgpu `MemoryHints::Manual { 64..128 MB }` in devlog 021, the egui version still showed substantially higher RSS than the iced version of ViewSkater on the same 4K dataset. A separate agent investigation of the iced repo (`../data-viewer`) revealed that the gap is **not** in wgpu configuration or texture upload path — both versions use identical APIs. The gap is in **cache architecture**: the egui version runs a double-tier cache (CPU LRU + GPU sliding window) while iced runs a single-tier cache (GPU only, decode-and-drop).

## Test data recap

| State | iced | egui (wgpu Manual 64/32) |
|-------|------|---------------------------|
| App launch | ~0 MB (display artifact) | 244 MB |
| After loading 4K dir (~140 frames) | 313 MB | 872 MB |
| Delta | +313 MB | +628 MB |

The "0 MB at launch" in iced is a cosmetic artifact — `CURRENT_MEMORY_USAGE` is initialized to `Mutex::new(0)` and read by the FPS overlay before the first sysinfo poll fires (gated behind a 1-second throttle and a window event). Both versions use the same `sysinfo::Process::memory()` RSS, polled once per second.

## What we hypothesized vs. what's actually true

| Hypothesis | Reality |
|---|---|
| iced uses a different memory metric | False — same `sysinfo` RSS |
| iced uses smaller cache window | False — same 11 slots (`cache_count=5`, `2n+1`) |
| iced uses better `MemoryHints` | **False — iced uses default `Performance`**, the same setting we tuned away from |
| iced uses a leaner texture upload path | False — same `queue.write_texture()` API |
| iced uses BC1 compression | False — defaults to `compression_strategy="none"`, same as us |
| iced is single-tier cache, no CPU LRU | **True — this is the actual difference** |

The most surprising finding: **iced uses worse `MemoryHints` than us and still uses ~315 MB less RSS**. This proves gpu_allocator block size is not where the gap lives — it's strong evidence that our `MemoryHints::Manual` tuning, while a valid 127 MB win, was not the primary lever.

## The real difference: cache architecture

### iced (single-tier)

`src/cache/gpu_img_cache.rs:25-74`

```rust
let img = cache_utils::load_original_image(...)?;
let rgba_image = img.to_rgba8();
let rgba_data = rgba_image.into_raw();  // owned Vec<u8>, ~33 MB for 4K
let texture = cache_utils::create_gpu_texture(&self.device, ...);
cache_utils::upload_uncompressed_texture(&self.queue, &texture, &rgba_data, ...);
Ok(CachedData::Gpu(texture.into()))
// rgba_data Vec dropped here as the function returns
```

What is retained: only `Arc<wgpu::Texture>`. The raw RGBA bytes go out of scope and are freed immediately after upload.

### egui (double-tier)

We run **two caches simultaneously**:

1. **`DecodeLruCache`** (`src/cache.rs`) — `HashMap<usize, egui::ColorImage>` on the CPU heap, capped by `lru_budget_mb` (default 1024 MB). This is the `L:` column in the FPS overlay. Decoded RGBA pixels stay resident in process RAM.
2. **`SlidingWindowCache`** (`src/cache.rs`) — `VecDeque<Option<egui::TextureHandle>>` for the 11-slot window. This is the `C:` column. On a discrete GPU the actual texture data is in VRAM; only the small handle bookkeeping is in RSS.

iced has only the second tier. No CPU LRU.

## Why we have a CPU LRU and iced doesn't

The `DecodeLruCache` exists because **slider scrubbing through high-resolution images is slow without it**. Decoding a single 4K PNG takes ~90 ms (devlog 019). Without an LRU, scrubbing the slider through 100 4K images would force ~9 seconds of decode work on the slider thread — unusable.

With the LRU, previously-displayed images are kept as decoded `ColorImage` in CPU memory. Re-displaying a cached image only requires uploading the (already-decoded) pixels to the GPU — ~1-2 ms instead of ~90 ms. Slider scrubbing becomes instant for cached images.

iced's slider may have a different design that doesn't require this. Or iced may simply accept slower slider scrubbing on high-resolution datasets. The investigation didn't measure iced's slider performance directly — that's an open question.

## Where the egui +628 MB delta actually goes

For the `L:180 C:45` test scenario (after some scrubbing, on Linux wgpu Manual):

- **180 MB** — `DecodeLruCache` itself (real CPU heap, fully tracked in `L:`)
- **45 MB** — `SlidingWindowCache` handle bookkeeping (logical, mostly in VRAM on discrete GPU)
- **~363 MB** — peak staging burst from 11 simultaneous `TextureHandle::set()` calls in a single frame. Each `ColorImage` (~33 MB) sits in egui's `TextureManager` as a pending `TextureDelta::Set` until the next frame's `egui_wgpu::Renderer::update_texture` consumes it. With 11 textures arriving in one frame, 363 MB of staging is alive simultaneously
- **~40 MB** — gpu_allocator host overhead under `Performance` hints (mostly recovered by our `Manual` tuning; some residual remains)
- **Remainder** — glibc arena retention. Linux glibc `malloc` does not aggressively `madvise(DONTNEED)` freed arenas. Once RSS has grown to N MB to hold peak staging, it typically stays at N MB even after the `ColorImage`s are dropped, until `malloc_trim(0)` is called or the allocator decides to release pages on its own

## Why the L:180 + 45 = 225 MB doesn't match the 628 MB delta

The FPS overlay's `(L:180 C:45)` parenthetical visually suggests these are sub-components of the RSS total. They aren't. On a discrete GPU:
- `L:` is a real subset of RSS (CPU heap)
- `C:` is logical GPU allocation, mostly in VRAM, not in RSS at all
- The remaining ~400 MB of growth is invisible in either column — it's egui's pending texture deltas, gpu_allocator overhead, and glibc arena retention

This is the UI clarity issue documented in devlog 021. The agent's recommendation to add `device.get_internal_counters().hal.memory.total_allocated_bytes` would split this into honest `RSS:` + `GPU:` numbers and would have caught the asymmetry immediately during diagnosis.

## Why glow shows much less RSS than wgpu

This is a question the iced investigation surfaces but doesn't directly answer. Our own analysis:

### glow (OpenGL) upload path

1. Background thread decodes image → `ColorImage`
2. Main thread calls `ctx.load_texture(name, color_image, ...)`
3. egui's `TextureManager` queues a `TextureDelta::Set`
4. Next frame: `egui_glow::Painter` consumes the delta and calls `glTexImage2D` directly with the pixel pointer
5. The OpenGL driver handles GPU upload internally — **no host-visible staging buffer is allocated in process memory**
6. `ColorImage` is consumed by the upload, freed

Any temporary buffers used by the driver for the upload live at the driver level, not in process RSS. On discrete GPUs the texture ends up in VRAM, also not in RSS.

### wgpu upload path

1-3. Same as glow up to the `TextureDelta::Set` queue
4. Next frame: `egui_wgpu::Renderer::update_texture` calls `queue.write_texture(...)`
5. `queue.write_texture` allocates an internal **staging buffer** in host-visible memory (CPU RAM, accessible by the GPU)
6. Pixel data is copied into the staging buffer
7. A buffer-to-texture copy command is recorded
8. At the next `queue.submit`, the GPU executes the copy
9. After the copy completes, the staging buffer is "released" — but gpu_allocator pools the underlying host memory blocks for reuse
10. `ColorImage` is consumed, freed

The staging buffer is **counted in process RSS** because it's host-visible memory mapped into the application's address space. gpu_allocator's pooling means that even after individual staging buffers are freed, the underlying blocks stay reserved.

### The key asymmetry

| | glow | wgpu |
|---|---|---|
| Texture pixels (final destination) | VRAM (off-RSS) | VRAM (off-RSS) |
| Staging during upload | Driver-managed (off-RSS) | gpu_allocator host blocks (in RSS) |
| Pending texture deltas | egui CPU memory (in RSS) | egui CPU memory (in RSS) |
| gpu_allocator overhead | None (no gpu_allocator) | Yes (in RSS) |

OpenGL drivers have decades of optimization for hiding upload mechanics from the application. wgpu makes the upload pipeline explicit and visible — every staging buffer is real memory in your process. This is by design (explicit modern APIs) but it means your app's apparent memory usage goes up even though the actual GPU work is the same.

The DecodeLruCache asymmetry **also** matters but only after scrubbing has populated the LRU. For the initial-load case (`L:0 C:348`), the entire gap between glow and wgpu is the upload path difference, not the LRU.

### Important: the staging buffer gap is not just visibility

The above framing makes the glow-vs-wgpu gap sound like pure accounting ("same RAM, different number"). It's actually **two things stacked**:

**1. Accounting (visibility)**

Staging buffers exist in RAM in both cases — that's physics on discrete GPUs, where the CPU cannot directly write to GPU-optimal memory layouts. The bytes have to land somewhere CPU-accessible before being copied into VRAM.

- **glow**: The OpenGL driver allocates the staging area through low-level mechanisms (driver-mapped DMA buffers, pinned/locked pages, mmap'd device file regions, kernel-managed scratch pools). These bytes exist in RAM but **are not counted in your process's RSS** by `sysinfo::Process::memory()` or `top`. They show up under "video memory" or kernel slab accounting instead
- **wgpu**: `gpu_allocator` allocates host-visible buffers through standard Vulkan/DX12 APIs. These end up in your process address space the same way any heap allocation does. **They are counted in RSS**

So the same physical RAM is used, but glow's portion is hidden from process memory accounting while wgpu's is visible.

**2. Efficiency (actual usage)**

Even if RSS were a perfect accounting metric, wgpu would still use **more total RAM** than glow:

- **OpenGL drivers** have decades of optimization for staging reuse. The driver maintains internal scratch buffers per GL context, reused across uploads. A few MB of scratch handles arbitrary upload throughput
- **`gpu_allocator`** pools released staging buffers, but its block sizes grow to fit the worst-case peak (the 11-simultaneous-upload burst we discussed). Once the pool has grown, it doesn't shrink. And `queue.write_texture()` doesn't try to reuse a single persistent staging area — each upload may get a fresh allocation

So wgpu's staging strategy is genuinely less efficient AND its bytes are visible to RSS, while glow's strategy is more efficient AND its bytes are invisible to RSS. Both axes contribute to the gap.

**What this means for fixing it**

- **The accounting half cannot be fixed** at our layer. It's how modern explicit graphics APIs work. wgpu allocates host buffers through its own allocator, which is owned by the application
- **The efficiency half can be fixed** but only by bypassing `egui_wgpu`'s default upload path. Options:
  - Use `wgpu::util::StagingBelt` — a ring buffer that recycles staging across frames
  - Persistent mapped staging buffer — allocate once, write into the same memory repeatedly
  - Both require dropping to direct wgpu rendering via `egui::PaintCallback` / `egui_wgpu::CallbackTrait` instead of using `egui::TextureHandle::set()`

This is why an atlas-style implementation is more compelling than just "atlas packs images more tightly" — an atlas forces you down the `PaintCallback` path anyway, which is exactly where you can fix the staging efficiency at the same time. A future atlas implementation should plan for both: bin-packed atlas layout AND a single persistent staging belt.

## The actual fix: GPU-side LRU + upload spreading

The earlier framing of "remove the LRU vs keep slider performance" was wrong. The right question is: **why is the LRU on the CPU at all?**

### Historical accident, not a design decision

The `DecodeLruCache` exists because the slider performance work (devlogs 008, 009) took the simplest possible fix: add a separate CPU-side cache of decoded `ColorImage`s. It worked, it was easy, it shipped. But the result is that we store the same pixel data **twice**:

- Once decoded in CPU heap (`DecodeLruCache`, ~1024 MB budget — the dominant RSS contributor)
- Once on the GPU when displayed (`SlidingWindowCache`, 11-slot window)

And the budget is on the **wrong cache** — the CPU LRU has the configurable budget, the GPU cache is fixed at 11 slots regardless of available memory.

### The fix: single GPU-side LRU

Replace both caches with a single budget-based GPU LRU:

```rust
GpuTextureLruCache {
    handles: HashMap<usize, egui::TextureHandle>,
    order: VecDeque<usize>,
    budget_bytes: usize,
    total_bytes: usize,
}
```

Behavior:
- LRU eviction by GPU memory budget (default e.g. 1024 MB)
- The "sliding window" becomes a prefetch hint: "load 5 forward + 5 back into this cache"
- All cached textures live on the GPU; nothing on CPU heap beyond `TextureHandle` bookkeeping (~few KB per texture)
- Slider scrubbing hits the GPU LRU directly with zero CPU upload — even faster than current
- Re-decode only happens for textures evicted from the GPU LRU
- For 4K images at 1024 MB: ~31 cached textures (vs current 11). For 1080p: ~128 textures cached.

This is **independent of the renderer choice** — it benefits glow and wgpu equally. Process RSS drops by ~600 MB on discrete GPUs because pixel data lives in VRAM (off-RSS) instead of CPU heap (in-RSS).

### Why upload spreading is also needed

Moving the LRU to the GPU fixes the **scrubbing** RSS gap (~600 MB). It does **not** fix the **initial-load** RSS gap (also ~600 MB). At initial load, both backends start with `L:0` — empty LRU. The initial-load gap comes from a different problem: **batched uploads in a single frame**.

When the sliding window cache loads its 11 textures at directory open:

1. 11 background threads decode images → produce `ColorImage`s
2. Main thread receives them and calls `ctx.load_texture()` for each — **all 11 in the same frame**
3. egui's `TextureManager` queues 11 `TextureDelta::Set` entries
4. Next frame: `egui_wgpu::Renderer::update_texture` processes all 11 deltas
5. For each one, `queue.write_texture()` allocates an internal staging buffer
6. **All 11 ColorImages + all 11 staging buffers are alive simultaneously: ~726 MB peak in CPU heap**

The data is then uploaded, ColorImages dropped, staging buffers released. But two memory pools refuse to give the pages back to the OS:

- **glibc malloc** — never `madvise(DONTNEED)`s freed arenas without an explicit `malloc_trim(0)`. Once the arena has grown to fit 11 simultaneous 33 MB allocations, it stays that size
- **gpu_allocator** — pools released staging buffer blocks for reuse. Once the pool has grown to hold 11 staging buffers, it doesn't shrink

So the data was freed, but the high-water mark shapes the steady state. RSS stays inflated.

### Key insight: the SlidingWindowCache itself is not the RSS bloat

Once the sliding window is populated, the actual texture pixel data is on the GPU (VRAM on discrete GPUs). Process RSS only carries small `TextureHandle` bookkeeping (~few KB). The cache itself is fine.

The 660 MB of RSS bloat is entirely from **transient stuff during the upload that should be gone but isn't**:

- The ColorImages used to upload — they were freed
- The staging buffers from `queue.write_texture()` — they were freed
- **The allocator pools that grew to hold them — these are still there**

Spreading uploads to 1-2 per frame means:
- gpu_allocator only ever needs to hold 1-2 staging blocks → pool stays small
- glibc only ever holds 1-2 simultaneous 33 MB allocations → arena reuses the same slot for each successive upload → arena stays small
- Steady-state RSS settles at ~50-100 MB above baseline instead of ~660 MB above baseline

This is exactly why iced uses ~313 MB despite uploading the same 11 textures. Each iced loader task is sequential, so glibc and gpu_allocator never see a high-water mark of more than 1 simultaneous upload.

### Why iced is so much smaller — the full picture

Both versions use the same wgpu API and the same `queue.write_texture()` upload path. The difference is **how many uploads are alive simultaneously**:

| | iced | egui (current) |
|---|------|----------------|
| Upload pattern | One async task at a time, sequential | All 11 deltas batched into one frame |
| Peak staging | ~33 MB (1 image) | ~363 MB (11 images) |
| Peak ColorImages alive | ~33 MB | ~363 MB |
| glibc arena high-water mark | ~33 MB | ~363 MB |
| gpu_allocator pool high-water mark | ~1 staging block | ~11 staging blocks |
| Steady-state RSS after load | ~313 MB | ~872-999 MB |

The CPU LRU contributes another ~180 MB **after scrubbing**, but at initial load the entire gap is the upload pattern, not the LRU.

### Renderer independence

Both fixes (GPU LRU + upload spreading) sit at the cache layer, **above** the renderer abstraction. They use `egui::TextureHandle` and egui's standard `TextureManager` flow, which is identical on glow and wgpu.

This means:

- **Switching renderers remains trivial after these fixes** — same small diff as today (drop the `wgpu` feature, remove `WgpuSetup::Existing` setup, drop `--renderer` flag and Graphics settings). ~50-100 LOC delete
- **Both backends benefit equally** — glow today also climbs to ~1.1 GB after scrubbing because of the same CPU LRU. Moving the LRU to the GPU drops glow's scrubbed RSS dramatically too
- **The wgpu-only switch decision is fully decoupled from the cache architecture** — they can be done independently in either order

This is important context for the user's question about whether the GPU LRU work locks us into wgpu. It doesn't.

### Combined impact: RSS budget after both fixes

Estimated RSS after implementing GPU LRU + upload spreading on Linux wgpu Manual:

| Component | Before | After | Notes |
|-----------|--------|-------|-------|
| Launch baseline | 244 MB | 244 MB | Structural wgpu/Vulkan runtime |
| CPU `DecodeLruCache` | 180 MB | 0 MB | Moved to GPU |
| Peak staging burst | 363 MB | ~66 MB | Spread to 1-2 per frame |
| glibc retention from burst | ~150 MB | ~10 MB | No high-water mark |
| gpu_allocator overhead | ~50 MB | ~30 MB | Smaller pool, fewer blocks |
| Other wgpu/egui retained state | ~150 MB | ~150 MB | Structural |
| **Total** | **~872 MB** | **~430-460 MB** | |

**iced reference: ~313 MB.** We'd land ~120-150 MB above iced.

### Can we fully match iced?

**Almost, but probably not exactly.** The remaining ~120-150 MB gap is structural egui retained state — `TextureManager` bookkeeping, retained shader/pipeline state, font atlases, dual-pane app state, etc. iced's equivalents exist but are smaller because iced is less general-purpose.

Closing that final gap would require changes to egui itself or moving away from egui's `TextureHandle` API entirely (custom wgpu rendering via `egui::PaintCallback`, similar to the atlas approach). High effort for ~150 MB.

**Practical answer:** With both fixes, we go from ~872 MB → ~430-460 MB. That's a ~50% RSS reduction. We're not exactly matching iced (313 MB), but we're in the same ballpark — well within "this is fine for an image viewer" territory.

### What requires a tradeoff (the worse alternative we'd avoid)

Listed for comparison; **not** what we'd do:

1. **Remove `DecodeLruCache` entirely** — what we considered before realizing the GPU LRU was the right answer. Saves ~180 MB CPU but kills slider scrubbing on revisit (~90 ms per frame for 4K)
2. **Switch to raw file bytes in the LRU** — iced's CPU strategy. Saves memory but you have to decode at draw time, which has the same scrubbing problem
3. **Lower default `lru_budget_mb`** — bandage. Doesn't fix the architectural issue

### Recommendation for the wgpu-only switch decision

Don't block the switch on full iced-parity. The cache architecture is independent of the renderer choice — the same memory issues exist on glow today, just less visibly. The wgpu switch is justified by:

- Cross-platform modern API
- macOS performance (Metal > deprecated OpenGL)
- eframe's own direction (0.34.0 made wgpu default)
- Linux/Windows perf parity after our `device.poll(Wait)` fix

What you want before switching is **observability** — adding GPU counters to the FPS overlay so you can actually see what's allocated. With that, you can verify the wgpu switch's memory characteristics in real time and tune separately.

The bigger memory wins (LRU resizing, upload spreading) are worth pursuing but on a separate branch dedicated to cache architecture, not coupled to the renderer choice.

## Recommended next steps (separate branch)

In priority order:

1. **Add GPU counter line to FPS overlay** — `RSS:872 GPU:340 (L:180 C:160)` format. Splits CPU heap from gpu_allocator memory. Cheap, immediate observability win, no architectural commitment
2. **Replace `DecodeLruCache` + `SlidingWindowCache` with single GPU-side LRU** — biggest memory win, no perf cost. The sliding window becomes a prefetch hint feeding the LRU. Both glow and wgpu benefit equally
3. **Spread `TextureHandle::set()` across frames** during initial cache fill — limit to 1-2 per frame. Prevents allocator high-water mark, keeps glibc/gpu_allocator pools small
4. **Add `malloc_trim(0)` after initial cache fill completes** — band-aid for any residual glibc retention. Already called after `close_images()`; would also call after batch loads
5. **Consider atlas approach later** if remaining ~150 MB residual matters — see devlog 008 analysis

These are cache architecture changes, not renderer changes. They should not block shipping the renderer-selection branch.

## Reference: iced source files

Located in the iced version of ViewSkater (separate repo at `../data-viewer`):

- `src/main.rs:1180-1264` — wgpu Instance/Adapter/Device creation (no `memory_hints` set)
- `src/main.rs:1270-1279` — `iced_wgpu::Engine::new` with atlas config
- `src/utils/mem.rs:11-81` — sysinfo-based RSS tracking
- `src/cache/gpu_img_cache.rs:18-75` — single-tier GPU cache (decode → upload → drop)
- `src/cache/cpu_img_cache.rs:18-31` — alternative CPU strategy (raw file bytes)
- `src/cache/cache_utils.rs:159-181` — `queue.write_texture()` upload site
- `src/config.rs:6` — `DEFAULT_CACHE_SIZE = 5` (same as us)
- `src/settings.rs:168-174` — defaults: `cache_strategy="gpu"`, `compression_strategy="none"`

Investigation summary written by a separate agent run, full report at `../data-viewer/tmp/iced_memory_investigation.md`.

## Key files

- `devlogs/019_memory_usage.md` — original memory investigation
- `devlogs/020_renderer_selection.md` — wgpu/glow renderer selection
- `devlogs/021_wgpu_frame_pacing.md` — wgpu frame pacing fix and `MemoryHints` tuning
- `src/cache.rs` — `DecodeLruCache` (CPU LRU) and `SlidingWindowCache` (GPU sliding window) — the two caches whose interaction creates the asymmetry vs iced
