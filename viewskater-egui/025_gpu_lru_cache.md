# 025 — Move DecodeLruCache from CPU heap to GPU

Follow-up to 024 (GPU counter) and the memory work deferred from the wgpu
switch (020–022). Replaces `DecodeLruCache`'s CPU-heap storage of decoded
`ColorImage`s with GPU-resident `TextureHandle`s. Slider-scrub revisits are
now zero-upload (LRU hit → no CPU→GPU copy), and the slider-scrub RSS that
previously reached 1.4 GB (one full pass) and kept climbing toward ~1.9 GB
under continued scrubbing on the 4K dataset now holds near ~930 MB.

Item #2 from the feat/gpu-cache plan. `SlidingWindowCache` intentionally
untouched (it was already GPU-backed).

## What ships

### `src/cache.rs::DecodeLruCache`

Struct shape changed:

```rust
// before
entries: HashMap<usize, egui::ColorImage>,  // CPU pixels

// after
entries: HashMap<usize, egui::TextureHandle>,  // GPU textures
ctx: egui::Context,                            // for load_texture
```

API-shape changes:

| Method | Before | After |
|---|---|---|
| `new` | `new(budget_mb)` | `new(ctx, budget_mb)` |
| `get` | `Option<&ColorImage>` | `Option<TextureHandle>` (cloned) |
| `insert` | `(idx, ColorImage)` | `(idx, name, ColorImage) -> TextureHandle` |
| `set_budget_mb`, `clear`, `len`, `total_mb` | unchanged | unchanged |

Key design choice: **LRU owns the upload**. `insert` consumes a decoded
`ColorImage`, calls `ctx.load_texture(...)`, stores the resulting handle,
and returns a clone so the caller can display it immediately. Callers
never touch `TextureHandle` themselves except to read from `get`. This
keeps the two "upload moments" (sliding window's `load_texture` inside
its `poll()` and LRU's `load_texture` inside `insert`) in one-for-one
symmetry — each cache manages its own GPU lifecycle.

Budget bookkeeping is `width × height × 4` computed at insert time,
mirroring the old `ColorImage` math and the `SlidingWindowCache::total_bytes`
formula. This is the *logical* byte count, not the real gpu_allocator
footprint (which is slightly larger due to block-suballocation). The two
agreed to within ~1 MB during 024's verification pass.

### `src/pane.rs::load_sync`

Was a ~70-line function with two upload branches (reuse `current_texture`
via `tex.set()` vs. `ctx.load_texture()` when `None`). Now ~35 lines with
one path:

```rust
// hit: no upload
if let Some(handle) = self.decode_cache.get(file_index) {
    self.current_texture = Some(handle);
    return;
}
// miss: decode → LRU uploads → point current_texture at returned handle
let handle = self.decode_cache.insert(file_index, name, color_image);
self.current_texture = Some(handle);
```

The `tex.set()` reuse trick is gone. It saved a GPU allocation per scrub
frame against a single reused texture slot, but that optimization was
incompatible with the GPU LRU — if the LRU wants to hand back a cached
handle, it has to be a *different* texture, not a mutated copy of the
current slot. Also: `tex.set()` mutated the underlying texture data
in-place via the id; if `current_texture` was ever a clone of a sliding
window slot (which `navigate()` makes it), `set()` would have corrupted
the sliding window's visual content. Removing that code path also
removes this latent sharp edge.

### `src/pane.rs::Pane::new`

Now takes `&egui::Context` as the first argument so the embedded
`DecodeLruCache` can be constructed. All three call sites already had a
context in scope:

- `src/app.rs::App::new` — `cc.egui_ctx`
- `src/app.rs::App::new` (pane1 branch) — same
- `src/app/handlers.rs::set_dual_pane` — function already takes `ctx`

## Behavior changes

1. **LRU hit is now free.** Previously: clone `ColorImage`, `tex.set()`
   → re-upload ~33 MB staging buffer per frame. Now: point at cached
   handle, zero upload. Log output on a revisit is simply
   `LRU hit [64]` — no decode/convert/upload columns.

2. **LRU eviction frees GPU allocations.** Dropping a `TextureHandle`
   queues a `TextureDelta::Free`; egui_wgpu processes it on the next
   render tick, and gpu_allocator reclaims the block.

3. **Double-backed entries are a real thing now, temporarily.** An image
   currently in `SlidingWindowCache` and also revisited via scrub lives
   as two independent `TextureHandle`s — one in each cache. Up to
   11 × 32 MB ≈ 360 MB of worst-case double-counting on 4K. Intentional
   tradeoff from the handoff doc: keeps the refactor small and leaves
   `SlidingWindowCache` untouched. A dedupe pass (`Arc<TextureHandle>`
   sharing, or having sliding-window `poll()` check LRU first) is
   possible future work.

4. **`current_texture` is no longer a stable slot.** Previously it held
   one long-lived handle that got rewritten. Now each navigation either
   points to a sliding-window slot, an LRU entry, or a fresh upload
   (miss path). Drop behavior is fine: clones keep textures alive as
   long as at least one reference exists; the last drop queues the
   GPU free.

## Verification

### Functional

On `data/demo/4k_PNG_10MB` (140 × 3840×2160 PNG), `cache_count=5`,
`lru_budget_mb=1024`:

- **Arrow-key navigation** — unchanged (sliding-window path only).
- **Slider scrub forward** → each frame shows correct image; on miss,
  debug log shows
  `load_sync [N]: decode=57ms convert=6ms upload=0.0ms [LRU: k / XYZ MB]`
  with `XYZ` climbing toward 1024.
- **Slider scrub back** through already-visited indices → `LRU hit [N]`
  every frame, no decode or upload time. Confirms the GPU cache returns
  the resident texture directly.
- **Budget eviction** → `[LRU: 32 / 1012 MB]` after 32 × 32 MB 4K textures,
  then holds there while older entries fall off.
- **Dual-pane** — opens and navigates; `ctx` plumbing works.

### Memory comparison (Linux, Balanced mode, RTX 3090)

Paired test: full-directory slider scrub through all 140 images, then
release. Measured via the new `RSS:` and `GPU:` columns.

|                                   | Before (#2 not applied) | After (#2 applied)     |
|-----------------------------------|-------------------------|------------------------|
| Launch                            | 252 MB                  | 251 MB                 |
| After opening 4K dir              | 873 MB                  | 872 MB                 |
| After one full scrub + release    | **1400 MB**, GPU 354 MB | **928 MB**, GPU 1.3 GB |
| After sustained scrubbing         | **~1900 MB**, GPU 354 MB| **~930 MB**, GPU 1.3 GB|

RSS saving: **~470 MB** on a single pass, **~1 GB** on sustained scrubbing
(where the old CPU LRU kept accumulating arena pressure through repeated
decode/upload cycles while the GPU LRU saturates at budget and stays there). The "after"
row matches the refactor's intent — pixels that previously lived in CPU
heap (as `ColorImage` in the LRU) now live in VRAM (as
`TextureHandle`-backed textures tracked by gpu_allocator). On a discrete
GPU (this 3090) the VRAM bytes are off-RSS, so the move converts
in-RSS bytes 1:1 to off-RSS bytes.

The GPU counter climbed to 1.3 GB = 353 MB sliding window + ~950 MB LRU,
which respects the 1024 MB LRU budget (with the double-counting overlap
noted above accounting for the difference).

### What this did *not* fix

Initial-load RSS (872 MB right after opening the directory, before any
scrub) is unchanged. That's the **batched-upload peak from
`SlidingWindowCache::initialize`** — 11 `load_texture` calls processed
in a single frame, peaking staging buffers at ~363 MB and leaving glibc
arena blocks pooled above. Item #3 (spread uploads across frames) is
still needed to close that gap; the prediction from devlog 022 is
872 → ~430–460 MB with both #2 and #3 landed.

## Gotchas

1. **`ctx.load_texture` name collisions are harmless.** Sliding window
   and LRU both use the file basename. Egui keys textures by generated
   ID, not name; the name is only used for debugging. No conflict.
2. **`insert` borrows `self.entries` mutably across an upload.** The
   implementation runs eviction *before* the `load_texture` call so
   the freed blocks are visible to gpu_allocator before it allocates
   the new one. Reverse order would briefly inflate the GPU high-water
   mark.
3. **`DecodeLruCache::clear` does not `malloc_trim`.** Explicit close
   goes through `App::close_images`, which already calls `malloc_trim(0)`
   on Linux. No extra trim needed on the cache side.
4. **Removed `tex.set()` means no per-frame reuse.** If we ever notice
   allocation thrash on scrubs with tiny images (<1 MB each) where the
   old reuse trick helped, we can reintroduce a single reusable slot
   for the miss path. For 4K the per-scrub allocation cost is in the
   noise vs. decode.
