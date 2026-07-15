# PR #31: File Discovery Options (Recursive, Hidden)

**PR**: https://github.com/ggand0/viewskater-egui/pull/31
**Author**: Fabian (BafDyce)
**Branch**: `feat/discovery-options`

## Summary

Adds two new file discovery options to settings: recursive directory search and include hidden files. Introduces an `ImageDiscoveryOptions` struct that wraps the existing `ImageSortOrder` along with `include_hidden: bool` and `recursive: bool`.

## Final state (pre-merge, 8 commits)

### Architecture

PR #24 introduced a split between saved sort defaults (`settings.image_sort_order`) and the active session sort (`current_sort`) so the View menu can temporarily override sort without saving. This PR extends that pattern to include recursive/hidden options.

The contributor's solution: keep `current_sort` as just `ImageSortOrder` for the View menu. Store recursive/hidden/default sort in `settings.image_discovery_options`. A helper merges them at call time:

```rust
fn current_discovery_options(&self) -> ImageDiscoveryOptions {
    let mut opts = self.settings.image_discovery_options;
    opts.sort_order = self.current_sort;
    opts
}
```

This avoids the two-copy sync problem entirely -- the View menu sort stays independent, recursive/hidden come from settings, and they're combined on the fly when a pane needs them.

### Changes

- **`src/settings.rs`**: New `ImageDiscoveryOptions` struct. Old `image_sort_order` removed from `AppSettings`, replaced by `image_discovery_options`. Settings UI sort ComboBoxes write to `image_discovery_options.sort_order`. Two toggle switches for recursive/hidden. Typo fixed ("Discover recursively"). `SettingsChanges.pane_settings` includes `image_discovery_options`.
- **`src/file_io.rs`**: `enumerate_images` takes `ImageDiscoveryOptions`. Private `enumerate_images_inner` handles recursive traversal and hidden file filtering. `compare_names` changed to compare full paths.
- **`src/app.rs`**: `current_sort` stays as `ImageSortOrder`. Settings modal handler syncs `current_sort` from settings and calls `reload_sorted_panes` on `pane_settings` change. Sort snapshot comparison restored in menu bar block.
- **`src/app/handlers.rs`**: `current_discovery_options()` helper merges `current_sort` with `settings.image_discovery_options`. All `open_path` calls use this helper. `reload_sorted_panes` uses `pane.dir_path` instead of deriving from current image, preserves active image with `jump_to`. Dual pane opening uses `pane.dir_path` instead of `image_paths[0].parent()`.
- **`src/menu.rs`**: Reset to Default reads from `settings.image_discovery_options.sort_order`. Footer shows relative path (strips `dir_path` prefix and leading slash).
- **`src/pane.rs`**: `dir_path: Option<PathBuf>` added to `Pane`. Set in `open_path` after successful enumeration. `open_path` takes `ImageDiscoveryOptions`.

## Review history

### Initial review (v1, 1 commit)

Recursive discovery worked after app relaunch but reload didn't work. Root cause: settings modal wrote to `settings.image_discovery_options` but `current_discovery_options` (the active copy) was never synced. Only the Reset to Default button in the Sort By menu bridged the two copies.

### Second review (v2, 3 commits)

Contributor fixed reload and added relative paths and `dir_path` on `Pane`. But broke sorting: removed sort snapshot comparison from menu bar block, and left two sort order fields in `AppSettings` (`image_sort_order` and `image_discovery_options.sort_order`) with the settings UI writing to the wrong one.

### Third review (v3, 8 commits)

Contributor fixed all flagged issues: restored sort, removed duplicate sort order, fixed leading slash, fixed typo, made `enumerate_images_inner` private. Also found and fixed a dual pane bug where opening a second pane used `image_paths[0].parent()` instead of `dir_path`.

## Post-merge TODO

- **`compare_names` full-path comparison**: Changed from `file_name()` to `as_os_str()`. In recursive mode this gives subdirectory grouping which makes sense. In non-recursive mode all files share the same parent so it doesn't matter in practice. But it's a behavioral change from the original sorting -- consider reverting to `file_name()` for non-recursive mode if users report unexpected sort order.
