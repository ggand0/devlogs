# Mini Spinner in Footer

## Overview
Replaced the centered overlay loading spinner with a compact mini spinner in the footer. The overlay was obstructing image viewing during background loading.

## Design Decision: Footer vs Menu Bar

**Footer (current choice):**
- Pros: Near the copy buttons and index display, keeps status info together in one area
- Cons: Hidden during fullscreen mode (footer hides unless cursor at bottom)

**Menu bar (alternative):**
- Pros: Always visible, could go left of FPS display
- Cons: Menu bar is already crowded with dropdown menus, separates loading status from image index

Chose footer because:
1. Loading spinner relates to current image navigation (shown next to index)
2. User can see spinner while looking at footer for position info
3. Menu bar real estate is limited

## Implementation

### Widget Changes

**circular.rs** - Added size customization:
```rust
impl Circular {
    /// Set the size of the spinner (default: 48.0)
    pub fn size(mut self, size: f32) -> Self {
        self.size = size;
        // Scale bar height proportionally (default is 4.0 for 48.0 size)
        self.bar_height = size / 12.0;
        self
    }
}

/// Create a small circular spinner for compact spaces like footer (18px)
pub fn mini_circular<'a, Message: Clone + 'a>() -> Element<...> {
    Circular::new().size(18.0).into()
}
```

### Footer Changes

**ui.rs** - Added `show_spinner` parameter to `get_footer()`:
```rust
pub fn get_footer(
    footer_text: String,
    metadata_text: Option<String>,
    pane_index: usize,
    show_copy_buttons: bool,
    show_spinner: bool,  // NEW
    options: FooterOptions,
    available_width: f32,
) -> Container<'static, Message, WinitTheme, Renderer>
```

Spinner element placed before copy buttons:
```rust
let spinner_element: Element<...> = if show_spinner {
    mini_circular()
} else {
    container(text("")).width(0).height(0).into()
};

row![
    spinner_element,      // NEW - left of copy buttons
    copy_filepath_button,
    copy_filename_button,
    mark_badge,
    coco_badge,
    text(state.footer_text)
]
```

### Removed Loading Overlay

**pane.rs** - Removed `loading_overlay` wrapping from `build_ui_container()`:
```rust
// BEFORE
loading_overlay(base_container, show_spinner)

// AFTER
base_container.into()
```

**ui.rs** - Removed `loading_overlay` from image widget wrapping.

## Footer Layout

```
┌─────────────────────────────────────────────────────────────────┐
│ 1920 x 1080 pixels  2.5MB          [⟳] [📁] [📄]     123/456   │
│ ← metadata                          ↑   ↑    ↑         ↑       │
│                                   spinner copy  copy   index   │
│                                          path  name            │
└─────────────────────────────────────────────────────────────────┘
```

The spinner (18px) appears left of the copy buttons when `loading_started_at` has been set for >1 second.

## Files Modified

- **src/widgets/circular.rs**: Added `size()` builder, `mini_circular()` helper
- **src/ui.rs**: Added `show_spinner` param to `get_footer()`, removed `loading_overlay` usage
- **src/pane.rs**: Removed `loading_overlay` wrapping

## Unused Code

After this change, the following are unused (compiler warnings):
- `circular()` function (48px version)
- `loading_overlay()` function

These could be removed or kept with `#[allow(dead_code)]` for future use.

## Future Considerations

1. **Horizontal array indicator**: Planned for showing sliding window cache status in footer
2. **Mini spinner next to array**: Could use `mini_circular()` alongside the array visualization
3. **Menu bar option**: If footer visibility in fullscreen becomes problematic, could add spinner to menu bar

## Dual-Pane Spinner Fix

### Bug

In dual-pane mode, the spinner clearing logic was checking the **global** loading queue state before clearing a pane's `loading_started_at`. This caused a bug where one pane's spinner would never clear if another pane finished loading later.

**Scenario:**
1. Load directory in pane 0 → `pane[0].loading_started_at = t0`
2. Load directory in pane 1 → `pane[1].loading_started_at = t1`
3. Pane 0 finishes → global queue still has pane 1's work → `pane[0].loading_started_at` NOT cleared
4. Pane 1 finishes → only `pane[1].loading_started_at` cleared
5. **Bug:** Pane 0's spinner shows forever

### Root Cause

```rust
// BEFORE (buggy)
let still_loading = !app.loading_status.loading_queue.is_empty()
    || !app.loading_status.being_loaded_queue.is_empty();
if !still_loading {
    if let Some(pane) = app.panes.get_mut(pane_index) {
        pane.loading_started_at = None;  // Only clears if GLOBAL queue empty
    }
}
```

The `loading_status` is shared across all panes, so checking global queue state for per-pane clearing was incorrect.

### Fix

Clear `loading_started_at` for the specific pane when its load operation completes, regardless of global queue state:

```rust
// AFTER (fixed)
// Clear loading timer for this pane
if let Some(pane) = app.panes.get_mut(pane_index) {
    pane.loading_started_at = None;
}
```

### Correct Behavior

| Location | Granularity | Behavior |
|----------|-------------|----------|
| Footer | Per-pane | Each pane's spinner independently reflects its own loading state |
| Menu Bar | Global | Single spinner shows if *any* pane is loading |
