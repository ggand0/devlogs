# Tabbed Settings UI

**Date**: 2026-06-14
**Branch**: `feat/tabbed-settings-ui`

## Motivation

The settings modal was a single scrollable list of 5 sections (Control, Files, Display, Graphics, Performance) with 12 total controls. As contributors add settings via PRs (e.g. PR #25 added reset zoom/pan, PR #31 adds recursive/hidden file discovery), the list grows and requires scrolling. Tabs organize settings into logical groups and scale better as more items are added.

## Design

Two tabs split along a clear semantic boundary:

- **General**: Control (mouse wheel zoom), Files (sort key, direction), Display (footer, FPS overlay, cache overlay, sync zoom/pan, reset zoom/pan on nav) -- 8 controls total, all UX preferences
- **Performance**: Graphics (GPU memory mode radio), Performance (cache size, LRU budget, decode threads sliders) -- 4 controls total, all system/GPU resource tuning

The tab bar is fully custom-painted using `allocate_exact_size` + `painter.text()`, following the same pattern as `accent_slider` and `gpu_memory_radio`. Active tab text uses `theme.accent` (teal) with a 2px underline; inactive tabs are gray 160; hover goes to white (matching `widgets.hovered.fg_stroke.color` from the theme visuals).

## Implementation

All changes are in `src/settings.rs`. No changes to `theme.rs`, `app.rs`, or `menu.rs`.

### New types and functions

- `SettingsTab` enum with `General` and `Performance` variants, plus `ALL` array and `label()` method
- `tab_bar()` -- custom-painted horizontal tab selector
- `render_general_tab()` -- extracted Control + Files + Display sections
- `render_performance_tab()` -- extracted Graphics + Performance sections

### Modal structure (inside the card Frame)

```
Title ("Preferences")
Separator
Tab bar
Separator
ScrollArea (fixed max_height, per-tab id_salt)
  match active_tab {
    General => render_general_tab()
    Performance => render_performance_tab()
  }
"Saved" indicator
```

### State management

Active tab is stored in egui temp data keyed by `Id::new("settings_active_tab")`. Persists across modal open/close within a session, resets to General on app restart. Each tab gets its own ScrollArea state via `id_salt(Id::new("settings_scroll").with(active_tab))`.

## Stable modal height

The two tabs have different content heights. Without intervention, the modal resizes when switching tabs because egui's `Area` auto-sizes to content.

### Failed approaches

1. **`set_min_height` inside scroll content**: Set a floor on the scroll area's inner UI min_rect. Technically stabilized the height but left visible empty padding at the bottom of the shorter tab.

2. **`allocate_space` padding inside scroll area**: Added invisible space after the tab content to match the tallest tab. Bug: `allocate_space` inserts `item_spacing.y` between the last content widget and the padding, making the padded tab a few pixels taller than the unpadded one. Each switch accumulated the error.

3. **`allocate_space` padding outside scroll area**: Same `item_spacing.y` problem, but between the scroll area widget and the padding widget in the card layout.

4. **`max_height` on ScrollArea with lazy height tracking**: Set `max_height(prev_max)` where `prev_max` is the tallest `content_size.y` seen so far, stored in temp data. Worked within a session (temp data persists across modal open/close), but broke on fresh launch: `prev_max` starts at 0, the default tab (General) sets it, and switching to Performance (taller) caused a resize because Performance had never been measured. This bug was masked during development because temp data from previous modal opens survived within the same process.

### Working solution

On the first frame the modal opens (`target_h == 0.0`), measure both tabs using invisible child UIs before rendering anything:

```rust
for tab in SettingsTab::ALL {
    let rect = Rect::from_min_size(
        ui.cursor().min,
        vec2(ui.available_width(), 10000.0),
    );
    let mut child = ui.child_ui_with_id_source(
        rect, *ui.layout(),
        ("settings_measure", tab as u8), None,
    );
    child.set_invisible();
    let mut tmp = settings.clone();
    match tab {
        SettingsTab::General => render_general_tab(&mut child, &mut tmp, theme),
        SettingsTab::Performance => render_performance_tab(&mut child, &mut tmp, theme),
    }
    target_h = target_h.max(child.min_rect().height());
}
```

Key details:
- `child_ui_with_id_source` creates a child UI that does not affect the parent layout
- `set_invisible()` sets an empty clip rect and disables input, so nothing is painted or interacted with
- A unique id source `("settings_measure", tab as u8)` avoids widget ID conflicts with the real tab
- `settings.clone()` prevents the measurement pass from modifying actual settings
- The measured `target_h` is stored in temp data and used as `ScrollArea::max_height` with `auto_shrink([false, false])` so the viewport is fixed from frame one

## Tab hover: why Label with Sense::click() doesn't work

The initial implementation used `egui::Label::new(text).sense(egui::Sense::click())` for tab buttons. This makes egui treat the label as an interactive widget, applying its built-in visual styles:

- `widgets.hovered.weak_bg_fill` (gray 60 background) on hover
- `widgets.active.bg_fill` (accent/teal filled rect) on click

The accent-colored rect fill on click occluded the tab text. Painting hover text over the label (via `painter.text()`) was a hack that didn't address the underlying active-state fill.

The fix: use `allocate_exact_size` with `Sense::click()` and paint text manually with `painter.text()`. No egui Label widget means no built-in interactive styling. The color is computed directly from `is_active` / `response.hovered()` state. This follows the same custom-paint pattern used by `accent_slider` and `gpu_memory_radio` elsewhere in the file.

## Deprecation: `child_ui_with_id_source`

The invisible tab measurement uses `ui.child_ui_with_id_source()`, which is deprecated in egui 0.31. The replacement is `child_ui` with `UiBuilder` which is a larger API change. Suppressed with `#[allow(deprecated)]` for now. Should be updated on the next egui version bump. The `set_visible(bool)` method is also deprecated in favor of `set_invisible()`, which is what we use.

## Refactoring: `section()` helper

The heading + framed content pattern repeated 5 times across both tab functions:

```rust
ui.label(RichText::new("Heading").size(14.0).color(theme.heading));
// optional subtitle
ui.add_space(N);
Frame::default().fill(theme.section_bg).corner_radius(6.0).inner_margin(10.0)
    .show(ui, |ui| { ... });
```

Extracted into a `section(ui, heading, subtitle, theme, content)` helper. The subtitle is `Option<&str>` since only Files and Performance sections have one. This also standardized the pre-frame spacing to 4px (Graphics and Performance previously used 2px). Inter-section spacing (`add_space(12.0)` between sections, `add_space(10.0)` after the last) stays in the tab functions since it depends on layout position.
