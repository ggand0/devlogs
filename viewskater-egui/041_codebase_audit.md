# 041 - Codebase audit: bugs, optimizations, refactoring

Full read-through of `main` (post PR #18 slider preview, post PR #33 tabbed
settings) looking for refactoring and optimization opportunities. Codebase is
~5k lines and in good shape overall; the findings below are one real settings
bug, two genuine performance wins, some small wasteful patterns, and
structural cleanups. Nothing urgent.

## Bugs

### B1. "Slider Preview Budget" setting has no effect until restart

`apply_settings_to_caches()` (`src/app/handlers.rs:156-175`) updates every
pane setting except `preview_budget_mb`, and `ThumbnailCache` has no budget
setter (its `preview_budget_mb` field at `src/cache.rs:22` is fixed at
construction). Changing the budget in Preferences does nothing: the live
`ThumbnailCache` keeps the old budget, and even reopening a folder uses the
stale `pane.preview_budget_mb` copied at `Pane::new`.
`SettingsChanges::between` (`src/settings.rs:332-338`) already flags the
change; it's just never applied.

Fix: add `ThumbnailCache::set_budget_mb()` (evict if over the new budget,
mirroring `DecodeLruCache::set_budget_mb` at `src/cache.rs:715`) and update
`pane.preview_budget_mb` in `apply_settings_to_caches`.

### B2. Stale settings-modal height when Slider Preview is toggled

Tab content height is measured once and cached forever in temp data
(`src/settings.rs:434-460`). The Performance tab gains a slider row when
`slider_preview` is on (`src/settings.rs:692-703`), so if the height was
measured while it was off, the row overflows into a scrollbar. Invalidate the
cached `settings_max_tab_h` when `slider_preview` changes.

## Performance

### P1. Every slider release does a redundant full synchronous decode

`apply_slider_release` (`src/pane.rs:318-331`) calls `cache.jump_to`, which
runs `initialize`, which always synchronously decodes the center image from
disk (`src/cache.rs:199-203`). At that moment `pane.current_texture` is
already showing exactly that image (loaded during the scrub via the sliding
window or the LRU). Releasing the slider on a 4K PNG blocks the UI thread for
~60-80 ms decoding an image that is already on the GPU, then replaces the
texture with an identical one. Same story for `Pane::jump_to` when
`decode_cache` holds the target index.

Fix: let `initialize` accept an optional pre-existing `TextureHandle` for the
center slot; the pane passes `current_texture` (slider release) or a
`decode_cache` hit (jump) instead of decoding.

### P2. ThumbnailCache re-decodes the same thumbnail repeatedly during hover

`current_thumbnail_for` (`src/cache.rs:78-96`) sends a decode request every
frame the hovered index is uncached, and the worker (`src/cache.rs:31-50`)
drains the queue to "latest" without knowing what's already cached or in
flight. Hovering steadily over an uncached index: frame 1 starts a decode,
frames 2..N queue more requests during the ~50 ms decode, and after the
result lands the worker decodes the very same index again (possibly several
times).

Fix: track `requested_idx: Option<usize>` in the cache and skip the send when
that index is already pending.

### P3. Settings auto-save writes YAML every frame during a slider drag

`show_settings_modal` saves whenever the snapshot differs
(`src/settings.rs:509-512`), so dragging Cache Size writes the config file at
~60 writes/sec. Save on drag release or debounce by ~300 ms of quiescence.

### P4. Footer stats the file every frame

`paint_pane_footer` calls `std::fs::metadata(path)` per pane per frame
(`src/menu.rs:420`), including during continuous repaint while skating. Only
a microsecond-level syscall, but caching the formatted size keyed by the
current path makes it free.

### P5. (Micro, optional) Opaque RGBA conversion pays for premultiplication

In `convert_image` (`src/decode.rs:39-51`), `Color32::from_rgba_unmultiplied`
does three multiplies per pixel. PNGs whose alpha channel is entirely 255
could take a memcpy-style path after a cheap opacity scan. Worth maybe
10-20 ms on 4K images; only bother if profiling shows the convert step
matters.

## Refactoring

### R1. `Pane` duplicates six settings fields that must be manually synced

`cache_count`, `lru_budget_mb`, `decode_threads`, `mouse_wheel_zoom`,
`reset_zoom_pan_on_navigation`, `preview_budget_mb` are copied into `Pane`
(`src/pane.rs:24-30`), giving `Pane::new` a 7-argument signature repeated at
four call sites, and `apply_settings_to_caches` has to mirror every field
(which is exactly how B1 happened). Pass a `&AppSettings` (or a small
`PaneConfig` derived from it) to `Pane::new` and the apply path so the next
setting addition is a one-place change.

### R2. `paint_nav_slider` has outgrown itself

It takes `panes: &mut [Pane]` just to reach pane 0's thumbnail cache
(`src/app.rs:174-272`), mixes slider input, painting, and preview-popup
orchestration, and is called with dummy preview args in independent mode.
Split into a pure slider widget plus a separate preview step (taking
`Option<&mut ThumbnailCache>` and the paths slice). Relatedly, the
rail/handle painting is duplicated almost line-for-line in `accent_slider`
(`src/settings.rs:46-78`); extract a shared `paint_slider_rail(rect, t,
accent)` helper.

### R3. Module splits

`cache.rs` is now three caches plus a loader plus tests in one 868-line file;
a `cache/` module dir (`sliding_window.rs`, `lru.rs`, `thumbnail.rs`) would
keep each cache's invariants readable in isolation. Same spirit for
`show_central_panel` (`src/app.rs:466-690`), where the ~200-line dual-pane
layout block could move into a helper.

## Notes for the jxl-support rebase

- The thumbnail worker calls `image::open` directly (`src/cache.rs:39`), not
  `file_io::open_image`. On `jxl-support` the JXL decoding hook is registered
  lazily inside `open_image`, so JXL thumbnails would only work because some
  `open_image` call happened to run first. Route the worker through
  `open_image`, or register the hook once in `main()`.
- `jxl-support`'s `poll()` re-queue logic (`push_front` + `break` when at the
  JXL limit) diverges from main's simpler skip-stale loop; that hunk needs
  care in the rebase. The branch predates the ThumbnailCache entirely.

## Deliberate non-finding

The lazy-decode architecture from devlog 027 (cache encoded bytes, decode on
demand) remains the only fundamental fix for the parallel-decode RSS spikes,
but mimalloc already contains the damage and the rewrite touches the whole
cache layer. Not worth doing until something forces it (e.g. low-memory
targets).

## PR plan

1. **ThumbnailCache fixes (B1 + P2).** Same struct, same file, both small;
   existing eviction tests in `cache.rs` give a home for new budget-setter
   coverage. B1 is the user-visible bug (a Preferences slider that silently
   does nothing); P2 rides along nearly for free.
2. **Skip redundant decode on slider release / jump (P1).** Biggest perf win;
   its own PR because it changes the `initialize` signature and the
   cache-seeding invariant.
3. **Pane settings plumbing refactor (R1).** After the bug fix, not before;
   it's the refactor that would have prevented B1 but shouldn't block it.
4. **Small polish (B2, P3, P4).** Tiny, low-risk, fine to bundle.
5. Later/optional: R2, R3 when they annoy; P5 only if profiling justifies it.
