# ML Feature Gating and Architecture Refactoring

**Date:** 2025-10-28
**Branch:** `feat/image-selection`
**Related:** `015_image_selection_feature.md`

## Overview

Refactored the ML image selection feature to improve code organization and eliminate repetitive conditional compilation blocks. All ML-related logic is now encapsulated in a dedicated `ml_widget` module, making the codebase cleaner and more maintainable.

## Motivation

After implementing the basic image selection feature, the code had several issues:
- **Repetitive cfg blocks**: 5 message variants each with `#[cfg(feature = "ml")]`
- **Scattered logic**: ML handling code mixed throughout `app.rs` (~80 lines)
- **UI complexity**: Micro cfg blocks in `ui.rs` footer functions
- **Poor separation of concerns**: Feature-specific code not isolated

This refactoring addresses these issues while keeping the ML feature experimental and feature-gated.

## Changes Made

### 1. Created `ml_widget` Module (`src/ml_widget.rs`)

**Purpose**: Single module to encapsulate all ML-related code.

**Components**:
- `MlMessage` enum: Groups all ML message variants
  ```rust
  pub enum MlMessage {
      MarkImageSelected(usize),
      MarkImageExcluded(usize),
      ClearImageMark(usize),
      ExportSelectionJson,
      ExportSelectionJsonToPath(PathBuf),
  }
  ```

- `mark_badge()`: Creates visual badge widgets (green "SELECTED", red "EXCLUDED")
- `empty_badge()`: Fallback for when ML features are disabled
- `handle_ml_message()`: Delegates all ML message handling logic (moved from `app.rs`)

**From implementation**: Automatic conversion to main `Message` type
```rust
impl From<MlMessage> for Message {
    fn from(ml_msg: MlMessage) -> Self {
        Message::MlAction(ml_msg)
    }
}
```

### 2. Consolidated Message Enum (`src/app.rs`)

**Before** (5 variants, 5 cfg attributes):
```rust
#[cfg(feature = "ml")]
MarkImageSelected(usize),
#[cfg(feature = "ml")]
MarkImageExcluded(usize),
#[cfg(feature = "ml")]
ClearImageMark(usize),
#[cfg(feature = "ml")]
ExportSelectionJson,
#[cfg(feature = "ml")]
ExportSelectionJsonToPath(PathBuf),
```

**After** (1 variant, 1 cfg attribute):
```rust
#[cfg(feature = "ml")]
MlAction(crate::ml_widget::MlMessage),
```

### 3. Delegated Message Handling (`src/app.rs`)

**Before** (80+ lines in `update()` method):
```rust
#[cfg(feature = "ml")]
Message::MarkImageSelected(pane_index) => {
    if let Some(pane) = self.panes.get(pane_index) {
        if pane.dir_loaded {
            let path = &pane.img_cache.image_paths[...];
            // ... 15 lines of logic
        }
    }
}
// ... repeated for all 5 message variants
```

**After** (6 lines, delegates to module):
```rust
#[cfg(feature = "ml")]
Message::MlAction(ml_msg) => {
    return crate::ml_widget::handle_ml_message(
        ml_msg,
        &self.panes,
        &mut self.selection_manager,
    );
}
```

### 4. Cleaned Up UI Code (`src/ui.rs`)

Introduced `FooterOptions` builder pattern to eliminate micro cfg blocks.

**Before**:
```rust
#[cfg(feature = "ml")]
{
    let mark = get_mark_for_pane(0);
    get_footer(footer_text, 0, mark, show_copy_buttons)
}
#[cfg(not(feature = "ml"))]
{
    get_footer(footer_text, 0, show_copy_buttons)
}
```

**After**:
```rust
let options = {
    #[cfg(feature = "ml")]
    { FooterOptions::new().with_mark(get_mark_for_pane(0)) }
    #[cfg(not(feature = "ml"))]
    { FooterOptions::new() }
};
get_footer(footer_text, 0, show_copy_buttons, options)
```

**FooterOptions struct**:
```rust
pub struct FooterOptions {
    pub mark_badge: Option<Element<'static, Message, WinitTheme, Renderer>>,
}

impl FooterOptions {
    pub fn new() -> Self { ... }

    #[cfg(feature = "ml")]
    pub fn with_mark(mut self, mark: ImageMark) -> Self { ... }

    pub fn get_mark_badge(self) -> Element<...> { ... }
}
```

### 5. Updated Keyboard Handlers (`src/app.rs`)

**Before**:
```rust
tasks.push(Task::done(Message::MarkImageSelected(pane_index)));
```

**After** (wraps in MlAction):
```rust
tasks.push(Task::done(Message::MlAction(
    crate::ml_widget::MlMessage::MarkImageSelected(pane_index)
)));
```

### 6. Fixed Compiler Warnings

Added `#[allow(dead_code)]` to public API methods that will be used later:

**config.rs**:
- `Config.cache_size` field (part of config struct)

**selection_manager.rs**:
- `SelectionState`: `selected_count()`, `excluded_count()`, `marked_count()` (used in tests)
- `SelectionManager`: `current_state_mut()`, `current_state()`, `mark_image()` (public API)

### 7. Removed Emoji Characters

Changed badge labels to avoid rendering corruption:
- ~~"✓ SELECTED"~~ → **"SELECTED"**
- ~~"✗ EXCLUDED"~~ → **"EXCLUDED"**

Colors still distinguish the badges (green for selected, red for excluded).

## Architecture Benefits

### Before Refactoring
```
app.rs (2000+ lines)
├── Message enum (scattered cfg blocks)
├── update() method (ML logic mixed in)
└── keyboard handlers (inline ML messages)

ui.rs
├── get_footer() (cfg-gated signatures)
└── build_ui functions (micro cfg blocks everywhere)
```

### After Refactoring
```
ml_widget.rs (180 lines) ← All ML code isolated here
├── MlMessage enum
├── mark_badge() UI component
├── handle_ml_message() business logic
└── From<MlMessage> for Message

app.rs (cleaner)
├── Message::MlAction wrapper
└── Delegates to ml_widget::handle_ml_message()

ui.rs (cleaner)
├── FooterOptions builder pattern
└── No micro cfg blocks in function bodies
```

## Technical Details

### Module Imports
```rust
// main.rs
#[cfg(feature = "ml")]
mod ml_widget;

// ml_widget.rs
use iced_winit::runtime::Task;
use crate::app::Message;
use crate::selection_manager::{ImageMark, SelectionManager};
use crate::pane::Pane;
```

### Delegation Pattern
The `handle_ml_message()` function takes:
- `ml_msg: MlMessage` - The ML-specific message
- `panes: &[Pane]` - Read-only access to panes for image info
- `selection_manager: &mut SelectionManager` - Mutable access for state changes

Returns `Task<Message>` to maintain iced's async architecture.

### Builder Pattern Benefits
`FooterOptions` provides:
- Single cfg block at call site (vs multiple micro blocks inside function)
- Clean API: `FooterOptions::new().with_mark(mark)`
- Fallback behavior: `get_mark_badge()` returns empty widget if no mark set

## Impact Metrics

**Lines of code reduced**:
- `app.rs`: ~100 lines cleaner (removed repetitive match arms)
- `ui.rs`: ~60 lines cleaner (removed micro cfg blocks)
- Added `ml_widget.rs`: 180 lines (net improvement in organization)

**Cfg block reduction**:
- Message enum: 5 cfg attributes → 1 cfg attribute
- Update method: 5 cfg blocks → 1 cfg block
- UI functions: ~8 micro cfg blocks → ~4 clean cfg blocks at call sites

## Testing

✅ **Build without ML features**:
```bash
cargo check
# Finished `dev` profile [optimized + debuginfo] target(s) in 1.08s
```

✅ **Build with ML features**:
```bash
cargo check --features ml
# Finished `dev` profile [optimized + debuginfo] target(s) in 1.41s
```

✅ **No warnings** (except third-party dependency `ashpd` future-incompat)

✅ **Functionality preserved**:
- Keyboard shortcuts work (S/X/U/Ctrl+E)
- Badges display correctly
- State persistence works
- Export to JSON works

## Future Work

This refactoring sets up clean architecture for future ML features:

**Easy to add**:
- Clear all selections: Add `ClearAllMarks` to `MlMessage` enum
- Selection statistics: Add method to `SelectionManager`, display in badge
- Batch operations: Add new message variants, handle in `ml_widget`

**All changes stay in `ml_widget.rs`** - no need to touch `app.rs` or `ui.rs`.

### Scaling to Multiple Feature Gates

Current `FooterOptions` pattern works well for 1-2 features, but if more feature-gated functionality is added (e.g., photo editing, COCO visualization), consider these patterns:

**Option 1: Feature-specific builders**
```rust
let mut options = FooterOptions::new();

#[cfg(feature = "ml")]
{
    options = ml_widget::apply_footer_options(options, pane, &selection_manager);
}

#[cfg(feature = "photo_editing")]
{
    options = photo_edit::apply_footer_options(options, pane, &edit_state);
}
```

**Option 2: Chained builder methods**
```rust
let options = FooterOptions::new()
    .with_ml_data(pane, &selection_manager)     // cfg-gated inside
    .with_edit_data(pane, &edit_state);         // cfg-gated inside
```

**Option 3: Complete delegation (recommended for 3+ features)**
```rust
// Move entire footer rendering to separate module
mod footer_builder;

let footer = if show_footer && pane.dir_loaded {
    footer_builder::build(app, pane_index)  // Handles all features internally
} else {
    container(text("")).height(0)
};
```

**Recommendation**: Current approach scales to 2-3 features. Beyond that, delegate entire footer/UI rendering to a separate module to keep `build_ui()` clean.

## Conclusion

Successfully refactored experimental ML feature to follow clean architecture principles. Code is now:
- **Maintainable**: Changes isolated to one module
- **Readable**: No scattered cfg blocks
- **Extensible**: Easy to add new ML features
- **Safe**: Feature-gated, doesn't affect normal builds

Ready to merge as 80% complete experimental feature while focusing on higher-priority COCO visualization work.
