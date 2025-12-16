# App.rs Refactoring

**Date:** 2025-11-02

## Overview
Refactored `src/app.rs` from 2,043 lines to 687 lines (66% reduction) by extracting code into focused, single-responsibility modules.

## Changes

### 1. Message Enum Extraction
- **Created**: `src/app/message.rs` (68 lines)
- Moved `Message` enum with all variants to dedicated file
- Fixed `FontLoaded` error type (changed from `font::Error` to `()` since unused)

### 2. Message Handler Extraction
- **Created**: `src/app/message_handlers.rs` (995 lines)
- Extracted all message handling logic from 955-line `update()` method
- Created categorized handler functions:
  - `handle_ui_messages()` - About, Options, Logs
  - `handle_settings_messages()` - Settings operations
  - `handle_file_messages()` - File operations
  - `handle_image_loading_messages()` - Image loading
  - `handle_slider_messages()` - Slider and navigation
  - `handle_toggle_messages()` - Toggle and UI controls
  - `handle_event_messages()` - Mouse, keyboard, file drops
- Created single entry point `handle_message()` that routes all messages
- Simplified `update()` from 83 lines of routing logic to 1 line

### 3. Keyboard Handler Extraction
- **Created**: `src/app/keyboard_handlers.rs` (373 lines)
- Extracted `handle_key_pressed_event()` (321 lines)
- Extracted `handle_key_released_event()` (52 lines)
- Moved `is_platform_modifier()` helper function
- Handles all keyboard shortcuts and navigation

### 4. Settings Widget Extraction
- **Created**: `src/app/settings_widget.rs` (94 lines)
- Extracted `RuntimeSettings` struct
- Created `SettingsWidget` to encapsulate:
  - Modal visibility state
  - Save status messages
  - Active tab tracking
  - Advanced settings input state
  - Runtime configuration
- Replaced 5 DataViewer fields with single `settings` field
- Added helper methods: `show()`, `hide()`, `is_visible()`, etc.
- **Updated**: `src/app/message_handlers.rs` - All settings field accesses
- **Updated**: `src/settings_modal.rs` - Settings UI field accesses
- **Updated**: `src/ui.rs` - Settings visibility checks

## Results

### Line Count Progression
1. Original: **2,043 lines**
2. After message handlers: **1,180 lines** (-863)
3. After message enum: **1,180 lines** (no change, just organization)
4. After keyboard handlers: **817 lines** (-363)
5. After message routing: **734 lines** (-83)
6. After settings widget: **687 lines** (-47)
7. **Final: 687 lines** (66% reduction)

### Module Breakdown
- `src/app.rs`: 687 lines - Core DataViewer struct, initialization, view
- `src/app/message.rs`: 68 lines - Message enum
- `src/app/message_handlers.rs`: 995 lines - Message handling logic
- `src/app/keyboard_handlers.rs`: 373 lines - Keyboard event handling
- `src/app/settings_widget.rs`: 94 lines - Settings state management
- **Total**: 2,217 lines (organized vs 2,043 monolithic)

## Benefits

1. **Maintainability**: Each module has clear, single responsibility
2. **Readability**: Easy to find and modify specific functionality
3. **Testability**: Handlers can be tested independently
4. **Extensibility**: Simple to add new message types or keyboard shortcuts
5. **Clean Architecture**: Follows Elm/Iced patterns with clear separation

## Technical Notes

- Preserved all functionality - zero behavior changes
- Made methods `pub(crate)` for module access where needed
- Used Deref/DerefMut pattern to expose RuntimeSettings on DataViewer
- Fixed compilation across all feature gates (`coco`, `ml`)
- All imports cleaned up, no warnings

## Commits
1. Extract message handlers to separate module
2. Extract Message enum to separate module
3. Extract keyboard handlers to separate module
4. Simplify update() by delegating to message handler
5. Extract settings into dedicated widget module
