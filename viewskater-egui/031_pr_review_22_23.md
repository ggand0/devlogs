# 031 -- PR #22 and #23 review (YelovSK)

Both PRs are from YelovSK (Patrik Hampel), a first-time contributor. Self-described
as having zero Rust experience, using AI for syntax. Both touch only `src/pane.rs`.

## PR #23 -- Use smooth scroll and align wheel zoom input

### The bug

Scroll zoom without Ctrl (mouse_wheel_zoom enabled) feels choppy compared to
Ctrl+scroll zoom. Both go through `zoom_image()` but read different inputs:

- **Ctrl+scroll**: egui consumes the scroll events and produces `zoom_delta()`,
  which internally reads `smooth_scroll_delta`. egui applies exponential
  smoothing over multiple frames before returning the value.
- **Scroll zoom without Ctrl**: our code reads `raw_scroll_delta.y`, which gives
  discrete wheel ticks with no smoothing. Each notch is a single jump.

The smoothness difference has nothing to do with Ctrl itself. It's just that
`zoom_delta()` happens to use the smoothed input, while our scroll path was
reading the raw input.

### Initial version (commit 020a5c4)

1. Switches `raw_scroll_delta` to `smooth_scroll_delta` so scroll zoom without
   Ctrl gets the same frame-by-frame interpolation.
2. Removes the `if scroll != 0.0` guard, since `(0.0 * speed).exp() = 1.0`.
3. Hardcodes `scroll_zoom_speed = 0.003` and rescales `zoom_delta()` (pinch)
   via `pinch.powf(0.003 / egui_zoom_speed)` to match.

### Review concern: speed choice

The contributor hardcoded 0.003 (carried over from the original code) and wrote
rescaling math to bring Ctrl+scroll down from egui's default 0.005 to match.
Both paths felt slow at 0.003 on testing. We asked to bump to 0.005, which
matches egui's default (`1.0 / 200.0` in
[`memory/mod.rs:335`](https://github.com/emilk/egui/blob/0.31.1/crates/egui/src/memory/mod.rs#L335)).

### Updated version (commit 7387cd0)

The contributor took a cleaner approach than just changing the constant:

1. Defines `SCROLL_ZOOM_SPEED = 1.0 / 200.0` as a constant in `app.rs`.
2. Sets it on egui's options at startup: `cc.egui_ctx.options_mut(|o| o.scroll_zoom_speed = SCROLL_ZOOM_SPEED)`.
   This means both egui's internal Ctrl+scroll path and our scroll path read
   from the same source.
3. In `pane.rs`, reads the speed from `ui.ctx().options(|o| o.scroll_zoom_speed)`
   instead of hardcoding.
4. Drops the `pinch_factor` rescaling entirely. Since both paths now use the
   same option value, `zoom_delta()` and our scroll factor are guaranteed to
   be in sync. No rescaling needed.
5. Still switches `raw_scroll_delta` to `smooth_scroll_delta` and removes the
   zero-check guard.

This is simpler and more correct. Changing the speed in the future only requires
editing the constant in one place.

### egui internals (for reference)

`smooth_scroll_delta` is documented on `InputState` in egui:
https://docs.rs/egui/latest/egui/struct.InputState.html

The smoothing logic lives in `egui/src/input_state/mod.rs` (egui 0.31.1). It
uses `exponential_smooth_factor(0.90, 0.1, dt)` to spread large discrete scroll
ticks across multiple frames. Small deltas (< 1.0) pass through unsmoothed.

When a zoom modifier (Ctrl/Cmd) is held, egui routes scroll events into a
separate `smooth_scroll_delta_for_zoom` accumulator and bakes it into the
`zoom_factor_delta` returned by `zoom_delta()`. The `scroll_zoom_speed` option
(default `1.0 / 200.0` = 0.005) controls the scaling.

Related egui issues:
- https://github.com/emilk/egui/issues/4777 (Ctrl+scroll zoom regression)
- https://github.com/emilk/egui/issues/7650 (scroll to zoom without Ctrl in Scene)

### Status

Updated version reviewed. Ready to merge.

---

## PR #22 -- Toggle actual size on double-click (+18 / -4)

### What it does

Double-click toggles between fit-to-screen and actual size.
Standard image viewer behavior (macOS Preview, IrfanView, XnView, etc.).

Currently double-click resets zoom and pan unconditionally. The PR replaces that
with a toggle:

- **At fit (zoom == 1.0)**: zoom to actual size (`actual_size_zoom = 1.0 / scale`).
  `scale` is `min(window_width / image_width, window_height / image_height)`,
  the factor used to fit the image in the window. Dividing by it reverses the
  shrink to show real pixels.
- **At any other zoom level**: reset to fit (zoom = 1.0, pan = zero). Same as
  old behavior.

### Zoom-toward-cursor pan

When zooming into actual size, the code pans so the spot under the cursor stays in
place:

```rust
self.pan = (hover_pos - available.center()) * (1.0 - self.zoom);
```

This matches the formula in `zoom_image()` (line 412) when starting from
zoom=1.0 and pan=zero. Verified correct.

### Small image case

If the image is smaller than the window, `actual_size_zoom < 1.0` (the image
gets smaller to show at real size). In this case the code skips cursor-tracking
and just centers the image (`pan = zero`). If the image perfectly fits the
window, `actual_size_zoom = 1.0` and double-click does nothing, which is
correct since fit and actual size are the same.

### The `is_fit_to_screen` check

Uses `(self.zoom - 1.0).abs() < f32::EPSILON`, so only exact fit triggers the
toggle to actual size. If you've scrolled even slightly, double-click resets to fit
instead. This is fine: the "already zoomed -> reset" behavior is preserved
for any manually-zoomed state.

### Other

`scale` computation moved from after the double-click block to before it (line
349), since the double-click logic now needs it. No behavior change for the
rest of the code.

### Status

Code reviewed. Behavior tested. Merged.

---

## PR #24 -- Add sorting options (+291 / -39)

### What it does

Adds sort options to settings: sort by name, modified date, created date,
file size, or extension, with ascending/descending direction. Settings
persist to `settings.yaml`. Changing sort re-enumerates all panes while
preserving current image, zoom, and pan.

Files touched: `app.rs`, `app/handlers.rs`, `file_io.rs`, `pane.rs`,
`settings.rs`, `theme.rs`.

### Sorting logic (file_io.rs)

Clean separation: `sort_paths` for name/extension (no metadata needed),
`sort_files` for date/size (reads metadata once per entry via `ImageFile::new`).
All sort modes fall back to `compare_names` as a tiebreaker via `.then_with()`.
Uses `natord::compare` for natural sorting (already used in the original code).

### Settings changes (settings.rs)

`SettingsChanges` struct replaces the old bool return from `show_settings_modal`.
Uses the existing `snapshot` for comparison instead of individual `prev_*`
values. Backdrop layer moved from `Foreground` to `Middle`, modal from
`Tooltip` to `Foreground` to fix dropdowns being hidden. Selection stroke
color set to dark gray for readable dropdown text.

### UX discussion: View menu vs settings

We asked the contributor to add a Sort By submenu to the View menu for
quick access, since most image viewers (IrfanView, Gwenview, gThumb,
Geeqie) put sort options in a menu, not buried in settings.

Sorting has two use cases:
1. Temporary: sort to find something, then stop caring
2. Persistent: always want a specific sort for your workflow

For case 1, persisting the View menu sort is problematic because you might
forget you changed it and the next session starts with an unintended order.
For case 2, having a persistent default in settings makes sense.

Final design: View menu sort is session-only (writes to `current_sort`,
resets to settings default on restart). Settings modal defines the persistent
default. "Reset to Default" in the View menu submenu resets to the user's
configured default from settings, not a hardcoded value.

### Design note: View menu writing to settings

The existing View menu toggles (Footer, FPS, Cache Overlay) write directly
to `settings` fields and persist via the snapshot comparison in `app.rs`.
This means the View menu is coupled to the persistence layer, which is
not ideal. Sorting avoids this by using a separate `current_sort` field,
but this creates extra plumbing (the field on App, a `MenuBarState` wrapper
struct to stay under clippy's argument limit, snapshot comparisons for both
settings and current_sort).

The root issue is that View menu items shouldn't write to settings at all.
The existing toggles persisting from the View menu is a pre-existing design
issue. Cleaning this up (separating session state from persisted settings
for all View menu items) is out of scope for this PR but worth addressing
later.

### Our local commits

1. Radio button accent flash fix in Sort By submenu (set
   `widgets.active.bg_fill` to transparent)
2. Hint text under Files section in settings ("Default sorting for newly
   opened folders")
3. Session-only View menu sorting with Reset to Default, bundled into
   `MenuBarState` struct

### Bug: tiebreaker ignores sort direction

#### Symptom

Sorting by modified date ascending, then switching to descending: the last
chunk of images stays in the original ascending order instead of reversing.
Tested with `/home/gota/ggando/rust_gui/data/demo/small_images/`. Every
file in that directory has the same modified date (Jun 5, 2020), which means
the date comparison returns `Equal` for all pairs and the tiebreaker decides
the entire order.

#### How the sorting code works

The sorting happens in `sort_files` (`file_io.rs:93-104`):

```rust
fn sort_files(
    entries: Vec<DirEntry>,
    sort_direction: SortDirection,
    compare: impl Fn(&ImageFile, &ImageFile) -> Ordering,
) -> Vec<PathBuf> {
    let mut images: Vec<ImageFile> = entries.into_iter().map(ImageFile::new).collect();
    images.sort_by(|a, b| {
        apply_sort_direction(compare(a, b), sort_direction)
            .then_with(|| compare_names(&a.path, &b.path))  // BUG: not wrapped
    });
    images.into_iter().map(|image| image.path).collect()
}
```

`sort_files` takes three arguments:
- `entries`: the list of files to sort
- `sort_direction`: ascending or descending (from the user's choice)
- `compare`: a comparison function passed in from the call site

The call site for modified date sorting (`file_io.rs:48-50`):

```rust
ImageSortKey::Modified => {
    sort_files(entries, sort_order.direction, |a, b| a.modified.cmp(&b.modified))
}
```

The third argument `|a, b| a.modified.cmp(&b.modified)` becomes the `compare`
parameter inside `sort_files`. It compares two files by their modified time.

#### Key functions

**`cmp`**: Rust's comparison method. Compares two values and returns
`Ordering`: `Less`, `Equal`, or `Greater`. `cmp` itself doesn't know about
ascending or descending. It just answers "which value is smaller?"
`a.modified.cmp(&b.modified)` returns `Less` if `a` was modified earlier.

**`Ordering`**: an enum with three values.
- `Less`: first item is smaller than second
- `Equal`: both are the same
- `Greater`: first item is larger than second

`sort_by` uses `Ordering` to arrange items. When it gets `Less`, it puts
`a` before `b`. When it gets `Greater`, it puts `b` before `a`.

**`apply_sort_direction`** (`file_io.rs:106-111`):

```rust
fn apply_sort_direction(ordering: Ordering, sort_direction: SortDirection) -> Ordering {
    match sort_direction {
        SortDirection::Ascending => ordering,
        SortDirection::Descending => ordering.reverse(),
    }
}
```

Ascending: returns the ordering as-is. Descending: calls `.reverse()` which
flips `Less` to `Greater` and vice versa. This is the only thing that makes
sorting respect the user's direction choice. `cmp` and `compare_names` don't
know about direction -- they always compare the same way for the same inputs.

**`compare_names`** (`file_io.rs:113-118`):

```rust
fn compare_names(a: &Path, b: &Path) -> Ordering {
    natord::compare(
        &a.file_name().unwrap_or_default().to_string_lossy(),
        &b.file_name().unwrap_or_default().to_string_lossy(),
    )
}
```

Compares two filenames using natural ordering (`natord::compare`). Natural
ordering means `img_2` comes before `img_10`, unlike alphabetical ordering
which would put `img_10` first (because "1" < "2"). `compare_names` has no
direction parameter. It always returns the same result for the same two
filenames.

**`then_with`**: a method on `Ordering`. Only runs the closure when the
ordering is `Equal`. If the ordering is already `Less` or `Greater`, it
skips the closure and returns as-is.

```
Equal.then_with(|| some_comparison)    // runs the closure
Less.then_with(|| some_comparison)     // skips, returns Less
Greater.then_with(|| some_comparison)  // skips, returns Greater
```

The name reads as: "then, with this closure, decide the order." The `_with`
suffix is a Rust convention meaning "lazy/deferred via a closure." It appears
throughout the standard library: `unwrap_or` vs `unwrap_or_else`, `or` vs
`or_else`, `then` vs `then_with`. The version without the suffix evaluates
immediately; the version with it only runs when needed. This avoids
unnecessary work (e.g., string comparison) when the primary comparison
already decided the order.

#### Why `compare_names` is used during modified date sorting

When sorting by modified date, some files can have the same date. For those
files, `a.modified.cmp(&b.modified)` returns `Equal`. `sort_by` can't decide
their order from dates alone. Without a tiebreaker, their order would depend
on whatever `std::fs::read_dir` returns, which isn't guaranteed to be
consistent between runs.

The contributor used `compare_names` as a tiebreaker: when dates are equal,
fall back to comparing filenames. Filenames are unique within a directory so
they always produce a definite, stable order.

In the test directory, ALL files have the same modified date (Jun 5, 2020),
so the date comparison returns `Equal` for every pair. The entire order is
decided by `compare_names`. This made the bug affect all files, not just a
subset.

#### The bug

The contributor wrapped `compare` (the date comparison) with
`apply_sort_direction` so it reverses in descending mode. But he didn't wrap
`compare_names` (the tiebreaker). He said this was intentional.

Before the fix:

```rust
apply_sort_direction(compare(a, b), sort_direction)   // wrapped, reverses
    .then_with(|| compare_names(&a.path, &b.path))    // NOT wrapped, always ascending
```

When sorting descending:
- Files with different dates: `apply_sort_direction` flips the result.
  Correct.
- Files with the same date: `compare` returns `Equal`, `then_with` runs
  `compare_names`. `compare_names("img_01", "img_02")` returns `Less`
  (01 before 02). This goes to `sort_by` as-is. `img_01` comes first.
  That's ascending. Wrong for descending mode.

#### The fix

Wrap the tiebreaker with `apply_sort_direction` too:

```rust
apply_sort_direction(compare(a, b), sort_direction)
    .then_with(|| apply_sort_direction(compare_names(&a.path, &b.path), sort_direction))
```

Now in descending mode: `compare_names("img_01", "img_02")` returns `Less`.
`apply_sort_direction(Less, Descending)` flips it to `Greater`. `sort_by`
puts `img_02` first. Correct.

Same fix applied in both `sort_files` (line 101) and `sort_paths` (line 88).

### Status

Code reviewed. View menu added. Sorting tiebreaker bug found and fixed
locally. Waiting on contributor response.
