# Image Selection/Exclusion Feature

**Date:** 2025-10-21
**Branch:** `feat/image-selection`

## Overview

Implemented a keyboard-driven image marking system for dataset curation. Users can now mark images as "selected" or "excluded" while browsing, with persistent JSON storage for use in external processing scripts (e.g., Python).

## Motivation

Need to quickly organize image datasets for training classifiers. This feature allows marking images with keyboard shortcuts and exporting the selection data as JSON for downstream processing.

## Implementation

### Core Components

#### 1. Selection Manager (`src/selection_manager.rs`)
- **ImageMark enum**: Three states (Unmarked, Selected, Excluded)
- **SelectionState struct**: Stores marks per directory with HashMap
- **SelectionManager**: Handles loading/saving state with JSON persistence
- **Storage location**: `~/.local/share/viewskater/selections/<dir_hash>.json`
- **Hash-based filenames**: Uses directory path hash for unique per-directory storage
- **Auto-save**: Immediately persists changes on every mark operation

#### 2. Keyboard Shortcuts
Added to `src/app.rs` in `handle_key_pressed_event()`:
- **S key**: Toggle selected state (green badge)
- **X key**: Toggle excluded state (red badge)
- **U key**: Clear mark for current image
- **Ctrl/Cmd+E**: Open file picker to export JSON

#### 3. Message Handlers
New message variants in `Message` enum:
- `MarkImageSelected(usize)` - pane index
- `MarkImageExcluded(usize)` - pane index
- `ClearImageMark(usize)` - pane index
- `ExportSelectionJson` - trigger file picker
- `ExportSelectionJsonToPath(PathBuf)` - perform export

#### 4. Visual Indicators (`src/ui.rs`)
- **Footer badges**:
  - Green "✓ SELECTED" badge for selected images
  - Red "✗ EXCLUDED" badge for excluded images
  - Hidden for unmarked images
- **get_footer()**: Updated signature to accept `ImageMark` parameter
- **build_ui()**: Helper closure `get_mark_for_pane()` retrieves current image mark
- Works in both single-pane and dual-pane layouts

### Data Persistence

#### Storage Format (JSON):
```json
{
  "directory_path": "/path/to/images",
  "marks": {
    "image1.jpg": "Selected",
    "image2.png": "Excluded",
    "image3.jpg": "Selected"
  }
}
```

#### Load/Save Behavior:
- **Auto-load**: When opening a directory, loads existing selections if available
- **Auto-save**: Marks saved immediately after each change (S/X/U keypress)
- **Export**: Ctrl+E allows saving to custom location for Python scripts

### Dependencies Added

```toml
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
```

Note: `dirs = "5.0"` was already present.

## Usage Workflow

1. Open directory with images in ViewSkater
2. Navigate through images with arrow keys (A/D or ←/→)
3. Press `S` to mark current image as selected (green badge appears)
4. Press `X` to mark current image as excluded (red badge appears)
5. Press `U` to clear the mark
6. Press `Ctrl+E` (or `Cmd+E` on Mac) to export selections to JSON
7. Use exported JSON in Python to copy/process selected or non-excluded images

## Technical Details

### Integration Points

**app.rs**:
- Added `SelectionManager` field to `DataViewer` struct
- Initialized in `DataViewer::new()`
- Load selection state in `initialize_dir_path()` after directory opens
- Keyboard handlers toggle marks and trigger saves
- Export handler uses `rfd::AsyncFileDialog` for file picker

**ui.rs**:
- Updated `get_footer()` signature: `(footer_text, pane_index, mark)`
- Added `get_mark_for_pane` closure in `build_ui()` to retrieve marks
- Updated all `get_footer()` call sites (single-pane and dual-pane modes)
- Updated `build_ui_dual_pane_slider2()` to accept mark getter closure

### Bug Fixes

**Issue**: Crash with `cosmic-text` panic about 0 line height
**Cause**: Empty text widget with `.size(0)` for unmarked images
**Fix**: Removed `.size(0)`, using container width/height=0 to hide instead

## Future Enhancements

Potential improvements (not implemented):
- Batch operations (mark all, clear all, invert selection)
- Selection statistics in footer (e.g., "5 selected, 2 excluded")
- Export modes built-in (copy selected, copy non-excluded) - currently JSON only
- Undo/redo for mark operations
- Keyboard shortcut hints overlay

## Testing

✅ Compiles successfully with `cargo check`
✅ No crashes when dropping directory
✅ Marks persist across app restarts
✅ Export to JSON works with file picker
✅ Visual badges display correctly in footer
✅ Works in both single-pane and dual-pane modes
