# Refactoring Plan: Extract Modules from main.rs

**Date:** 2026-03-09
**Target branch:** `main` (after merging `feat/slider-performance`)
**X post:** https://x.com/gtgando/status/2030819669184282869

## Repo history cleanup

Needed to strip `Co-Authored-By: Claude` footers from 8 commits before making the repo public. Used `git filter-repo` to rewrite commit messages in-place, which changes hashes but preserves author/committer dates and all other metadata.

Since the rewrite invalidates commit SHAs referenced by PR #1 on GitHub, reproduced the repo from scratch:

1. Renamed the old GitHub repo to `viewskater-egui-v0`
2. Created a fresh empty `viewskater-egui` repo
3. Pushed main up to the pre-merge commit: `git push origin e0ecbaf:refs/heads/main`
4. Pushed the feature branch tip: `git push origin 266662a:refs/heads/feat/slider-performance`
5. Created PR #1 via `gh pr create`
6. Pushed full main (including the merge commit): `git push origin main --force`

Step 6 auto-closed PR #1 as "Merged" because GitHub detected the PR branch's commits became reachable from main via the merge commit. This is the same as merging locally and pushing, rather than clicking "Merge pull request" on the GitHub UI.

`filter-repo` also leaves behind git replace refs (in `.git/refs/replace/`) that map old commit hashes to new ones. These are local only and never pushed, but they clutter gitk with `replace/<hash>` labels. Cleaned up with `git replace -l | xargs -I{} git replace -d {}`.

## Motivation

`main.rs` is at 737 lines and contains PaneState, App, image conversion, CLI args, all UI rendering, and `main()`. As features grow (settings, more widgets, additional pane types), this will become harder to navigate. Extract logical units into separate modules now before the codebase grows further. Also fix all clippy warnings as part of the cleanup.

## Current file layout

| File | Lines | Contents |
|------|-------|----------|
| `main.rs` | 737 | Everything: PaneState, App, image conversion, CLI, UI, main() |
| `cache.rs` | 472 | SlidingWindowCache, SliderLoader, DecodeLruCache, debug overlay |
| `file_io.rs` | 53 | Path resolution, image enumeration |
| `perf.rs` | 85 | FPS overlay |

## Proposed modules

### 1. `src/pane.rs` (~210 lines)

Extract `PaneState` struct and its full impl:
- Struct definition with fields: `image_paths`, `current_index`, `current_texture`, `zoom`, `pan`, `cache`, `slider_loader`, `decode_cache`
- `new()`, `open_path()`, `load_sync()`
- Navigation: `navigate()`, `jump_to()`, `can_navigate_forward/backward()`, `is_next_cached()`, `poll_cache()`

### 2. `src/image_view.rs` (~70 lines)

Extract image rendering and zoom/pan interaction:
- `show_content()` and `show_image()` (currently methods on PaneState)
- `MIN_ZOOM`, `MAX_ZOOM` constants
- Zoom (scroll + pinch), pan (drag), reset (double-click)
- Display rect computation and `painter.image()` call

### 3. `src/decode.rs` (~45 lines)

Extract `image_to_color_image()`:
- The CICP bypass conversion (DynamicImage → ColorImage)
- Handles ImageRgb8, ImageRgba8, and fallback variants
- Future home for decode utilities (downscaled preview, format-specific decoders)

### 4. `src/app.rs` (~310 lines)

Extract `App` struct and its impl:
- `App` struct: `panes`, `perf`, `divider_fraction`
- `new()`, `handle_keyboard()`, `handle_dropped_files()`, `update_title()`
- `show_bottom_panel()` (slider UI + throttle + cache chain)
- `show_central_panel()` (single/dual pane layout, divider interaction)
- `eframe::App` impl (`update()`)

### 5. `main.rs` stays minimal (~30 lines)

- `mod` declarations
- CLI `Args` struct
- `main()` function

## Dependency flow

```
main
  → app
    → pane
      → cache
      → decode
      → file_io
    → image_view (used by pane for rendering)
    → perf
```

No circular dependencies. Each module has a clear single responsibility.

## Result

| File | Contents | ~Lines |
|------|----------|--------|
| `main.rs` | Args, main(), mod declarations | ~30 |
| `app.rs` | App struct, keyboard/DnD/slider/panel UI | ~310 |
| `pane.rs` | PaneState, navigation, loading | ~210 |
| `image_view.rs` | Image rendering, zoom/pan | ~70 |
| `decode.rs` | image_to_color_image | ~45 |
| `cache.rs` | Unchanged | 472 |
| `file_io.rs` | Unchanged | 53 |
| `perf.rs` | Unchanged | 85 |

## Future-proofing

This structure supports planned features without further restructuring:
- **Settings/preferences** → new `settings.rs` module, used by `app.rs`
- **Additional viewer widgets** (histogram, metadata panel) → new modules, called from `app.rs`
- **Multiple viewer backends** (e.g., wgpu direct rendering) → swap `image_view.rs`
- **Faster decode libraries** → extend `decode.rs` with alternative backends
- **Thumbnail strip** → new module, integrated into `app.rs` panel layout

## Outcome

Merged `show_content`/`show_image` into `pane.rs` instead of a separate `image_view.rs` since they directly mutate PaneState's zoom/pan fields. Splitting them would require passing mutable references back and forth for no real benefit.

| File | Lines | Contents |
|------|-------|----------|
| `main.rs` | 36 | Args, main(), mod declarations |
| `app.rs` | 363 | App struct, keyboard/DnD/slider/panel UI |
| `pane.rs` | 302 | PaneState, navigation, loading, zoom/pan, image rendering |
| `decode.rs` | 44 | image_to_color_image |
| `cache.rs` | 473 | SlidingWindowCache, SliderLoader, DecodeLruCache |
| `file_io.rs` | 53 | Path resolution, image enumeration |
| `perf.rs` | 85 | FPS overlay |

Clippy warnings fixed:
- `&PathBuf` → `&Path` in `spawn_load` and `decode_sync`
- `map_or(false, ...)` → `is_some_and(...)` in cache and pane
- `#[allow(dead_code)]` on `DecodeResult::decode_ms` (kept for future per-image decode timing)
