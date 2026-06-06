# PR #31: File Discovery Options (Recursive, Hidden)

**PR**: https://github.com/ggand0/viewskater-egui/pull/31
**Author**: Fabian (BafDyce)
**Branch**: `feat/discovery-options`
**Base**: pre-0.2.0 (needs rebase onto main)

## Summary

Adds two new file discovery options to settings: recursive directory search and include hidden files. Introduces an `ImageDiscoveryOptions` struct that wraps the existing `ImageSortOrder` along with `include_hidden: bool` and `recursive: bool`.

### Changes across 6 files

- **`src/settings.rs`**: New `ImageDiscoveryOptions` struct. `AppSettings.image_sort_order` renamed to `image_discovery_options`. Two toggle switches added to settings UI.
- **`src/file_io.rs`**: `enumerate_images` takes `ImageDiscoveryOptions` instead of `ImageSortOrder`. New `enumerate_images_inner` handles recursive traversal and hidden file filtering. `compare_names` changed from `file_name()` to `as_os_str()` for full-path sorting.
- **`src/app.rs`**: `current_sort` renamed to `current_discovery_options` throughout.
- **`src/app/handlers.rs`**: All `open_path` calls updated to pass `current_discovery_options`.
- **`src/menu.rs`**: Sort menu accesses `current_discovery.sort_order`. Footer changed to show full path (with a TODO for relative path).
- **`src/pane.rs`**: `open_path` signature updated.

## Reload Bug and Fix

The contributor noted that reload on settings change didn't work.

### Root cause: two copies of discovery state

PR #24 introduced a split between two variables for sort order:
- `self.settings.image_discovery_options` -- the saved default, persisted to disk via the settings modal
- `self.current_discovery_options` -- the active state for the current session, mutated by the View > Sort By menu

This split is intentional for sort order: you can temporarily change sort from the View menu without overwriting your saved preference. The only bridge between them is the Reset to Default button in the Sort By menu, which replaces the entire `current_discovery_options` with `settings.image_discovery_options`.

The contributor added `recursive` and `hidden` to the same `ImageDiscoveryOptions` struct that contains `sort_order`. This bundles two separate concerns (sort order vs file discovery) into one struct. Because of the two-copy split, toggling recursive in the settings modal updates the saved default, but `current_discovery_options` never picks it up. All pane operations (`open_path`, `reload_sorted_panes`) use `current_discovery_options`, so `recursive` never takes effect.

### Why sort menu clicks don't help but Reset to Default does

The View > Sort By menu and the Reset to Default button both go through the same reload block in `app.rs`:

```rust
if self.current_discovery_options != discovery_snapshot {
    self.reload_sorted_panes(ctx);
}
```

But they modify `current_discovery_options` differently:

- **Sort key/direction click**: `current_discovery_options.sort_order.key = new_key` -- changes one field. `recursive` stays false.
- **Reset to Default**: `*current_discovery = settings.image_discovery_options` -- replaces the entire struct with the copy from settings, which includes `recursive: true`.

In the test flow below, after dropping demo/ with `recursive=false`, the pane finds 0 images and `image_paths` becomes empty. From that point, `reload_sorted_panes` is a no-op for both sort changes and Reset to Default, because it needs a path from `image_paths` to work with:

```rust
let Some(path) = pane.image_paths.get(pane.current_index).cloned() else {
    continue; // empty, skip
};
```

Neither sort changes nor Reset to Default actually reload panes. The difference only shows on the next redrop: sort changes left `recursive=false` on `current_discovery_options`, so the redrop enumerates without recursion and finds 0 images again. Reset to Default set `recursive=true`, so the redrop enumerates recursively and finds all images.

### Reproduction

1. Open `/data/ggando/data/demo/4k_PNG_10MB` (140 images)
2. Enable recursive in settings modal -- saved to disk, but `current_discovery_options` stays `recursive: false`
3. Drop `/data/ggando/data/demo` -- `open_path` uses `current_discovery_options` (recursive=false), finds 0 images at demo root, stale image still displayed
4. Change sort via View > Sort By -- block fires but `reload_sorted_panes` skips (pane empty), `recursive` still false. Redrop demo/ -- same, 0 images
5. Click Reset to Default -- block fires but `reload_sorted_panes` skips (pane empty), but now `current_discovery_options` has `recursive=true`. Redrop demo/ -- enumerates recursively, 8877 images

### Fix

Three lines in `src/app.rs`. After the settings modal runs, check if `current_discovery_options` diverged from `settings.image_discovery_options`. If so, sync and reload:

```rust
if self.current_discovery_options != self.settings.image_discovery_options {
    self.current_discovery_options = self.settings.image_discovery_options;
    self.reload_sorted_panes(ctx);
}
```

No changes to `settings.rs` needed. The contributor's `SettingsChanges.pane_settings` change is harmless (it triggers `apply_settings_to_caches` which just updates cache params).

## Remaining Issues

- **`compare_names` regression**: Changed from `file_name()` to `as_os_str()`, which alters sorting even in non-recursive mode.
- **Footer shows absolute path**: Should show path relative to source directory. Needs a `source_dir` stored somewhere (`Pane` is the right place).
- **Reload from subdirectory**: If already in recursive mode viewing a subdir image, `reload_sorted_panes` calls `open_path` with the current image path, and `resolve_path` extracts its parent (the subdir) as the base. Separate from the settings modal bug.
- **UI typo**: "Discovery recursively" should be "Discover recursively".
- **`enumerate_images_inner` is `pub`**: Should be private.
- **Settings migration**: Rename from `image_sort_order` to `image_discovery_options` silently resets existing users' sort preferences.
- **Needs rebase**: Branch is based on pre-0.2.0.
