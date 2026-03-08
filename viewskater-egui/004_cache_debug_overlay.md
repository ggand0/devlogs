# 004: Cache Debug Overlay

**Date:** 2026-03-07

## Goal

Add a visual overlay to inspect the sliding window cache state in real time — which slots are loaded, which are in-flight, and where the current position sits in the window.

## Implementation

### Overlay rendering (`cache.rs` — `show_debug_overlay`)

Uses `egui::Window` anchored to top-left, non-interactable, with semi-transparent dark background. Same pattern as the FPS overlay in `perf.rs`.

Layout:
```
┌──────────────────────────────────────────┐
│ Cache [10–20]                            │
│ [██][██][██][██][██][▶█][██][██][░░][░░]  │
│  10  11  12  13  14  15  16  17  18  19   │
│ ● Loaded  ● Loading  ● Empty             │
└──────────────────────────────────────────┘
```

### Cell states and colors

Each of the 11 slots maps to a file index (`first_file_index + slot_index`). Three states:

| State | Condition | Color |
|-------|-----------|-------|
| Loaded | `slots[i].is_some()` | Green (`#4CAF50`) |
| Loading | `in_flight.contains(file_index)` | Amber (`#FFB74D`) |
| Empty | Neither loaded nor in-flight | Dark gray (`#3C3C3C`) |
| Out of bounds | `file_index >= num_files` | Near-black (`#191919`) |

The current image's cell gets a white border (`StrokeKind::Outside`, 2px).

### Drawing approach

Uses `ui.allocate_exact_size()` to reserve a fixed rect for the cell grid, then paints directly via `ui.painter()`:
- `rect_filled` for cell backgrounds
- `rect_stroke` for current position border
- `painter.text()` for file index labels below cells

This gives pixel-level control over positioning without fighting egui's layout system. Cell dimensions: 28×20px with 2px gaps.

### Legend

`legend_swatch()` helper draws a small colored square inline with a text label using `allocate_exact_size(8×8)` + `rect_filled` + `ui.label()`. Three items in a `ui.horizontal` row.

### Integration

Called from `App::update()` after the FPS overlay:
```rust
if let Some(cache) = &self.cache {
    cache.show_debug_overlay(ctx, self.current_index, self.image_paths.len());
}
```

### egui 0.31 API note

`Painter::rect_stroke` in egui 0.31 requires a 4th argument `StrokeKind` (`Inside`, `Middle`, or `Outside`). Used `StrokeKind::Outside` so the border doesn't overlap the cell fill.

## Observation: bursty loading behavior with 4K images

With 4K images (decode ~50-100ms each), holding the arrow key reveals a bursty pattern:

1. Navigate through 5 cached images instantly (all green in overlay)
2. Stall — cache is exhausted, sync fallback blocks the UI per image
3. Meanwhile, 5 background threads that were spawned (one per navigate step) all finish around the same time since they were spawned within milliseconds of each other
4. Next `poll()` uploads all 5 textures at once → all cells flip green simultaneously
5. Navigate through that batch quickly, stall again, repeat

This differs from iced viewskater which produces a consistent ~11.5 FPS flow during held-key navigation. The egui version shows ~15-16 FPS on average but the actual rendering is uneven — fast bursts separated by stalls rather than steady throughput.

The root cause: OS key repeat fires faster (~33Hz) than 4K decode time (~10 FPS), so the user outruns the cache. The 5 preloaded images are consumed in ~150ms, then there's a ~500ms wait while the next batch decodes. All threads finish near-simultaneously because they were spawned at nearly the same time.

Potential approaches to address this (not yet implemented):
- **Rate-limited navigation** ("skate" mode): cap navigation speed to background decode throughput so the user never outruns the cache
- **Priority loading**: when navigating, cancel/deprioritize far-ahead loads and prioritize the immediate next image
- **Continuous key-hold detection**: bypass OS key repeat and drive navigation from the frame loop, pacing it to cache availability
