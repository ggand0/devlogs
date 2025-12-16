# ML Feature Architecture Summary

**Date:** 2025-10-28
**Purpose:** Quick reference for extending ML features (e.g., COCO visualization)

## Quick Overview

The ML dataset curation feature is **feature-gated** behind the `ml` cargo flag and follows a **module encapsulation pattern** where all ML-specific code lives in dedicated modules.

```bash
# Enable ML features
cargo run --features ml
```

## Architecture Pattern

### Core Principle: Complete Encapsulation

All ML-related code is isolated in feature-specific modules:

```
src/
├── ml_widget.rs           # ML UI, messages, and logic
├── selection_manager.rs   # State management for image marks
└── app.rs                 # Delegates to ml_widget (minimal coupling)
```

**Key idea**: `app.rs` knows almost nothing about ML internals. It just delegates.

### How It Works

#### 1. Messages (Enum Wrapper Pattern)

**ml_widget.rs:**
```rust
#[derive(Debug, Clone)]
pub enum MlMessage {
    MarkImageSelected(usize),
    MarkImageExcluded(usize),
    ClearImageMark(usize),
    ExportSelectionJson,
    ExportSelectionJsonToPath(PathBuf),
}
```

**app.rs (Message enum):**
```rust
#[cfg(feature = "ml")]
MlAction(crate::ml_widget::MlMessage),  // Single wrapper variant
```

**Benefit**: Only 1 cfg attribute in app.rs instead of 5+ scattered variants.

#### 2. Message Handling (Delegation Pattern)

**app.rs (update method):**
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

**ml_widget.rs:**
```rust
pub fn handle_ml_message(
    ml_msg: MlMessage,
    panes: &[Pane],
    selection_manager: &mut SelectionManager,
) -> Task<Message> {
    match ml_msg {
        MlMessage::MarkImageSelected(pane_index) => { /* ... */ }
        // ... all business logic here
    }
}
```

**Benefit**: 80+ lines of ML logic moved from app.rs to ml_widget.rs.

#### 3. Keyboard Handling (Optional Return Pattern)

**app.rs (handle_key_pressed_event):**
```rust
_ => {
    // Check if ML module wants to handle this key
    #[cfg(feature = "ml")]
    if let Some(task) = crate::ml_widget::handle_keyboard_event(
        key, modifiers, &self.pane_layout, self.last_opened_pane
    ) {
        tasks.push(task);
    }
}
```

**ml_widget.rs:**
```rust
pub fn handle_keyboard_event(
    key: &keyboard::Key,
    modifiers: keyboard::Modifiers,
    pane_layout: &PaneLayout,
    last_opened_pane: isize,
) -> Option<Task<Message>> {
    match key.as_ref() {
        Key::Character("s") | Key::Character("S") => {
            Some(Task::done(Message::MlAction(MlMessage::MarkImageSelected(...))))
        }
        _ => None  // Not an ML key
    }
}
```

**Benefit**: Returns `Option<Task>` so other features can coexist. Non-ML builds compile out entirely.

#### 4. UI Components (Builder Pattern)

**ui.rs (FooterOptions):**
```rust
pub struct FooterOptions {
    pub mark_badge: Option<Element<'static, Message, WinitTheme, Renderer>>,
}

impl FooterOptions {
    pub fn new() -> Self { Self { mark_badge: None } }

    #[cfg(feature = "ml")]
    pub fn with_mark(mut self, mark: ImageMark) -> Self {
        self.mark_badge = Some(crate::ml_widget::mark_badge(mark));
        self
    }
}
```

**Usage in build_ui():**
```rust
let options = {
    #[cfg(feature = "ml")]
    { FooterOptions::new().with_mark(get_mark_for_pane(0)) }

    #[cfg(not(feature = "ml"))]
    { FooterOptions::new() }
};
get_footer(text, index, show_buttons, options)
```

**Benefit**: Single cfg block at call site instead of scattered throughout function.

## State Management Pattern

### SelectionManager (Persistent State)

**src/selection_manager.rs:**
```rust
pub struct SelectionManager {
    current_state: Option<SelectionState>,
    data_dir: PathBuf,  // ~/.local/share/viewskater/selections/
}

impl SelectionManager {
    pub fn load_for_directory(&mut self, dir_path: &str) -> Result<(), std::io::Error>
    pub fn save(&mut self) -> Result<(), std::io::Error>
    pub fn toggle_selected(&mut self, filename: &str)
    pub fn get_mark(&self, filename: &str) -> ImageMark
}
```

**Integration in app.rs:**
```rust
pub struct DataViewer {
    // ... other fields
    #[cfg(feature = "ml")]
    pub selection_manager: SelectionManager,
}
```

**Lifecycle:**
- **Load**: When opening directory (`initialize_dir_path`)
- **Save**: Immediately after each mark change (S/X/U keypress)
- **Access**: Via `&mut self.selection_manager` in message handlers

## Adding New ML Features (e.g., COCO Visualization)

### Option 1: Extend Existing ml_widget Module

If COCO is closely related to image marking:

```rust
// ml_widget.rs
pub enum MlMessage {
    // Existing...
    MarkImageSelected(usize),

    // New COCO features
    LoadCocoAnnotations(PathBuf),
    ToggleBoundingBoxes(bool),
    ToggleSegmentationMasks(bool),
}
```

### Option 2: Create Separate coco_widget Module (Recommended)

If COCO is a distinct feature:

```rust
// src/coco_widget.rs
pub enum CocoMessage {
    LoadAnnotations(PathBuf),
    ToggleBoundingBoxes(bool),
    SelectCategory(String),
}

pub fn handle_coco_message(...) -> Task<Message> { /* ... */ }
pub fn render_annotations(...) -> Element<...> { /* ... */ }
```

**app.rs:**
```rust
#[cfg(feature = "ml")]
MlAction(crate::ml_widget::MlMessage),

#[cfg(feature = "coco")]  // Could share "ml" feature or be separate
CocoAction(crate::coco_widget::CocoMessage),
```

**Benefit**: Keeps COCO parsing/rendering separate from selection logic.

### Option 3: Unified ML Feature with Sub-modules

```
src/
├── ml/
│   ├── mod.rs           # Re-exports and delegation
│   ├── selection.rs     # Image selection (current ml_widget)
│   ├── coco.rs          # COCO annotations
│   └── visualization.rs # Shared rendering utilities
```

## Key Integration Points

### 1. Message Handling (app.rs update method)
Add delegation for new feature messages.

### 2. Keyboard Shortcuts (app.rs handle_key_pressed_event)
Add `handle_keyboard_event` call or extend existing ml_widget handler.

### 3. State Management (DataViewer struct)
Add feature-gated state manager:
```rust
#[cfg(feature = "coco")]
pub coco_manager: CocoAnnotationManager,
```

### 4. Initialization (DataViewer::new)
Initialize state when feature enabled:
```rust
#[cfg(feature = "coco")]
coco_manager: CocoAnnotationManager::new(),
```

### 5. UI Rendering (ui.rs)
Extend `FooterOptions` or create overlay elements:
```rust
#[cfg(feature = "coco")]
pub fn render_coco_overlay(annotations: &[Annotation]) -> Element<...>
```

## Common Patterns

### Pattern 1: Conditional Import
```rust
#[cfg(feature = "ml")]
use crate::selection_manager::SelectionManager;
```

### Pattern 2: Conditional Field
```rust
pub struct DataViewer {
    #[cfg(feature = "ml")]
    pub selection_manager: SelectionManager,
}
```

### Pattern 3: Conditional Message Variant
```rust
pub enum Message {
    #[cfg(feature = "ml")]
    MlAction(crate::ml_widget::MlMessage),
}
```

### Pattern 4: Conditional Handler
```rust
match message {
    #[cfg(feature = "ml")]
    Message::MlAction(ml_msg) => { /* delegate */ }
}
```

## Testing Strategy

### Build Verification
```bash
# Without features
cargo check

# With ML features
cargo check --features ml

# With all features (future)
cargo check --all-features
```

### No Scattered CFG Blocks
✅ **Good**: One cfg at enum variant, one at match arm
```rust
#[cfg(feature = "ml")]
MlAction(MlMessage),

// ... later
#[cfg(feature = "ml")]
Message::MlAction(ml_msg) => { /* ... */ }
```

❌ **Bad**: Scattered micro-blocks
```rust
#[cfg(feature = "ml")]
match x {
    #[cfg(feature = "ml")]
    Foo => { /* ... */ }  // Redundant nested cfg
}
```

## Performance Considerations

### Zero-Cost Abstraction
- Feature-gated code compiles out completely when disabled
- No runtime overhead for checking flags
- Dead code elimination at compile time

### Lazy Loading
```rust
// Load COCO annotations only when needed
#[cfg(feature = "coco")]
if let Some(coco_path) = &pane.coco_annotations_path {
    coco_manager.load_for_image(coco_path)?;
}
```

## Current ML Features

### Image Selection/Exclusion
- **Keys**: S (select), X (exclude), U (clear), Ctrl+E (export)
- **Storage**: `~/.local/share/viewskater/selections/<hash>.json`
- **Format**: JSON with `{marks: {"img.jpg": "Selected"}}`
- **Visual**: Green "SELECTED" / Red "EXCLUDED" badges in footer

### Files Modified (Merged)
- `src/ml_widget.rs` - 244 lines (messages, UI, logic)
- `src/selection_manager.rs` - 314 lines (state persistence)
- `src/app.rs` - Minimal delegation (~15 lines of ML code)
- `src/ui.rs` - FooterOptions pattern
- `Cargo.toml` - Feature flag `ml = []`

## Next Steps for COCO Visualization

### Recommended Approach
1. **Create `coco_widget.rs`** module following same pattern as `ml_widget.rs`
2. **Define `CocoMessage` enum** for COCO-specific actions
3. **Create `CocoAnnotationManager`** for parsing/caching COCO JSON
4. **Add delegation** in app.rs (1 message variant, 1 match arm)
5. **Render overlays** in ui.rs or pane rendering pipeline

### Key Questions to Answer
- Share `ml` feature flag or create `coco` flag?
- Render annotations in existing pane view or overlay?
- Load COCO JSON per-directory or per-image?
- UI for toggling annotation visibility?

### Suggested File Structure
```
src/
├── ml_widget.rs           # Existing selection feature
├── coco_widget.rs         # NEW: COCO messages and UI
├── coco_parser.rs         # NEW: COCO JSON parsing
├── annotation_renderer.rs # NEW: Bounding box/mask rendering
└── app.rs                 # Add CocoAction delegation
```

## References

- **Implementation details**: `devlogs/image-selection-feature_102125.md`
- **Refactoring notes**: `devlogs/ml-feature-refactoring_102825.md`
- **COCO format spec**: https://cocodataset.org/#format-data

---

**TL;DR for Next Agent:**

The ML feature uses a **module encapsulation pattern**. To add COCO visualization:
1. Create `coco_widget.rs` with `CocoMessage` enum
2. Add `#[cfg(feature = "coco")] CocoAction(CocoMessage)` to app.rs Message enum
3. Delegate handling: `Message::CocoAction(msg) => coco_widget::handle_coco_message(msg, ...)`
4. Follow same patterns as `ml_widget.rs` - keep app.rs minimal, isolate all COCO logic in module

All code compiles out when feature disabled. Zero runtime cost.
