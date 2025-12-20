# Devlog 032: Async Initial Loading Investigation

**Date:** 2025-12-20
**Status:** 🔴 Blocked - State Management Issue
**Branch:** `feat/async-initial-load`

## Problem Statement

When dropping large JP2 files (10-85MB HiRISE Mars imagery), the UI freezes for 10+ seconds during initial load because all images in the cache window (2×cache_count+1 = 11 images) are loaded synchronously. This creates a poor user experience where:
- UI is unresponsive for extended periods
- User cannot interact with the first image until all cache neighbors load
- No visual feedback during loading

**Goal:** Load only the first/dropped image synchronously, then load the cache window neighbors asynchronously in the background (sequential loading).

---

## Implementation Approach

### Phase 1: Planning (Completed)
Created comprehensive plan in `.claude/plans/whimsical-jumping-clock.md`:
- Identified current synchronous blocking flow
- Designed sequential loading pattern similar to existing `load_remaining_images()`
- Planned self-triggering message chain for sequential loads

### Phase 2: Core Implementation (Completed)
**Key Files Modified:**
- `src/pane.rs` - Modified to load only first image synchronously
- `src/cache/cpu_img_cache.rs` - Added `load_single_image()` method
- `src/cache/gpu_img_cache.rs` - Added `load_single_image()` method
- `src/cache/img_cache.rs` - Added trait method for single image loading
- `src/app.rs` - Added state tracking fields: `pending_initial_loads`, `is_initial_loading`
- `src/app/message.rs` - Added messages: `InitialLoadStarted`, `ContinueInitialLoad`

**Sequential Loading Flow:**
```
1. User drops file/directory
   ↓
2. Pane::initialize_dir_path() loads ONLY first image synchronously
   ↓
3. DataViewer::initialize_dir_path() calls load_initial_neighbors()
   ↓
4. load_initial_neighbors():
   - Generates interleaved indices (closest neighbors first: +1, -1, +2, -2...)
   - Loads first neighbor async
   - Stores remaining neighbors in pending_initial_loads
   - Sets is_initial_loading = true
   ↓
5. When first neighbor completes (LoadPos):
   - Checks is_initial_loading flag
   - If true, sends ContinueInitialLoad message
   ↓
6. ContinueInitialLoad handler:
   - Pops next image from pending_initial_loads
   - Starts async load
   - Returns Task that triggers next ContinueInitialLoad when done
   ↓
7. Repeat until pending_initial_loads is empty
```

### Phase 3: Bug Discovery - Message Ordering Race (Fixed)
**Issue:** First implementation used `Task::batch([async_task, InitialLoadStarted])` which executed concurrently.

**Timeline:**
```
05:22:41.898711Z LoadPos: is_initial_loading=false, pending_count=0  ← Completes FIRST
05:22:41.898722Z InitialLoadStarted: Storing 4 pending initial loads  ← Arrives SECOND
```

**Root Cause:** Tasks in `Task::batch()` execute concurrently, so `InitialLoadStarted` could arrive after the first `LoadPos` completed. When `LoadPos` checked `is_initial_loading`, it was still `false`, so continuation didn't trigger.

**Fix Applied (Commit: f33885a):**
Changed `load_initial_neighbors()` to set state synchronously before returning:
```rust
// CRITICAL: Set state IMMEDIATELY (synchronously) before returning
*pending_initial_loads = remaining;
*is_initial_loading = true;
```

This ensures `is_initial_loading=true` BEFORE the async task completes.

---

## Current Blocking Issue

### Symptom
Logs show state is set correctly but mysteriously clears before first async completion:

```
05:32:20.879725Z load_initial_neighbors: Set is_initial_loading=true, pending_count=4
05:32:20.879730Z AFTER load_initial_neighbors: is_initial_loading=true, pending_count=4 ✓
                                               ↓ (~2.7 seconds later)
05:32:23.566748Z LoadPos: is_initial_loading=false, pending_count=0  ✗
```

**Key Observation:** State is confirmed set to `(true, 4)` immediately after `load_initial_neighbors()` returns, but becomes `(false, 0)` by the time the first `LoadPos` completes.

### Investigation Status

**What We Know:**
1. ✅ State IS being set correctly (confirmed by logs at app.rs:347)
2. ✅ Mutable references work correctly (state persists after function returns)
3. ❌ State gets cleared between setting and first LoadPos completion
4. ❌ No "User navigation detected" logs (keyboard handlers not involved)
5. ❌ No evidence of `reset_state()` being called

**Potential Causes Under Investigation:**
1. **Hidden state reset:** Something is calling `pending_initial_loads.clear()` that we haven't found
2. **Multiple app instances:** Message handlers might be operating on different instances (unlikely in Iced)
3. **Async timing issue:** Some other message/handler is interfering
4. **Memory corruption:** State being overwritten (very unlikely in Rust)

**Places State Is Modified:**
```rust
// Setting to true:
- navigation_slider.rs:506 - Our synchronous set (CONFIRMED WORKING)
- message_handlers.rs:398  - InitialLoadStarted handler (shouldn't be called now)

// Setting to false:
- message_handlers.rs:416  - ContinueInitialLoad completion (shouldn't fire yet)
- keyboard_handlers.rs:222 - Left arrow navigation (not pressed)
- keyboard_handlers.rs:322 - Right arrow navigation (not pressed)

// Clearing pending_initial_loads:
- keyboard_handlers.rs:221 - Left arrow navigation (not pressed)
- keyboard_handlers.rs:321 - Right arrow navigation (not pressed)
```

### User Experience
- ✅ First image shows immediately after synchronous load
- ✅ First neighbor (image 1) loads successfully
- ❌ Sequential chain breaks - no continuation to image 2+
- ❌ User can navigate to image 1 but images 2+ remain unloaded

---

## Debugging Artifacts

### Comprehensive Logging Added
**app.rs:267** - Entry to initialize_dir_path
**app.rs:333** - Before calling load_initial_neighbors
**app.rs:347** - After load_initial_neighbors (shows state WAS set correctly)
**navigation_slider.rs:454** - Entry to load_initial_neighbors
**navigation_slider.rs:466** - Interleaved indices generated
**navigation_slider.rs:501** - Remaining neighbors count
**navigation_slider.rs:507** - State set confirmation
**message_handlers.rs:309** - LoadPos received
**message_handlers.rs:320** - State check when LoadPos completes (shows WRONG state)
**message_handlers.rs:390** - InitialLoadStarted (shouldn't fire)
**message_handlers.rs:404** - ContinueInitialLoad entry
**message_handlers.rs:411** - Each sequential load

### Test Dataset
**Path:** `/home/gota/ggando/rust_gui/data/jp2/hirise_mars_jp2/`
**Files:** 22 large JP2 files (HiRISE Mars imagery)
**File Sizes:** 10-85MB each
**Dimensions:** Up to 9512×35661 pixels (auto-resized to fit 8192×8192 GPU limit)
**Cache Settings:** cache_count=5 (11 total images in window)

---

## Next Steps

### Immediate Actions Needed
1. **Add state change tracing:** Log EVERY assignment to `is_initial_loading` and `pending_initial_loads` with stack trace or caller info
2. **Check for hidden clears:** Search for any vector operations that might clear pending_initial_loads
3. **Verify no state copies:** Ensure we're not inadvertently working with a cloned DataViewer instance
4. **Add watchpoint:** Consider using a debug build with memory watchpoints on the state fields

### Alternative Approaches If Current Fails
1. **Message-based state:** Move state into the message system instead of DataViewer fields
2. **Atomic operations:** Use Arc<Mutex<>> for state to ensure thread safety
3. **Global static:** Use a global static for sequential loading state (hacky but would prove the issue)
4. **Different triggering mechanism:** Instead of checking state in LoadPos handler, have a dedicated coordinator task

---

## Code References

### Key Functions
- `DataViewer::initialize_dir_path()` - [app.rs:266-350](src/app.rs#L266-L350)
- `load_initial_neighbors()` - [navigation_slider.rs:437-516](src/navigation_slider.rs#L437-L516)
- `handle_image_loading_messages()` - [message_handlers.rs:282-443](src/message_handlers.rs#L282-L443)
- `Pane::initialize_dir_path()` - [pane.rs:438-715](src/pane.rs#L438-L715)

### Sequential Loading Infrastructure
- State fields: [app.rs:125-126](src/app.rs#L125-L126)
- Messages: [message.rs:87-90](src/app/message.rs#L87-L90)
- LoadPos handler: [message_handlers.rs:308-328](src/message_handlers.rs#L308-L328)
- ContinueInitialLoad handler: [message_handlers.rs:403-441](src/message_handlers.rs#L403-L441)

---

## Related Devlogs
- [030_jpeg2000_support.md](030_jpeg2000_support.md) - Context for large JP2 file handling
- [031_footer_metadata_display.md](031_footer_metadata_display.md) - Recent footer work that runs in parallel

---

## Lessons Learned

### What Worked
✅ **Synchronous state setting** - Avoids message ordering races
✅ **Comprehensive logging** - INFO level logs critical for debugging async flows
✅ **Sequential pattern** - The ContinueInitialLoad message chain is sound design

### What Didn't Work
❌ **Task::batch() for state + async** - Creates race conditions
❌ **Assuming message order** - Messages arrive non-deterministically
❌ **Insufficient logging initially** - Should have added INFO logs from the start

### Still Unknown
❓ **Why state clears** - The core mystery remains unsolved
❓ **Iced message handling** - Need deeper understanding of Iced's update cycle
❓ **State lifetime** - Is DataViewer being copied somewhere we don't know about?

---

## Performance Goals (When Fixed)

**Target Behavior:**
- First image visible in <1 second after drop
- UI responsive immediately
- Cache window fills in background over 10-30 seconds
- User can navigate to adjacent images as they load
- No blocking operations in UI thread

**Current Blocking Time:** ~11 seconds for first interaction
**Target Blocking Time:** <1 second
**Improvement:** ~90% reduction in perceived load time

---

**Status Summary:** Implementation is architecturally sound, but blocked by mysterious state clearing issue. State is confirmed set correctly but disappears before first async callback. Needs deeper investigation into Iced's update cycle or potential hidden state modifications.
