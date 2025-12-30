# Async Directory Enumeration (Issue #73)

**Date**: 2025-12-29
**Branch**: `fix/async-dir-enum`
**Issue**: [#73 - Performance drops significantly when opening a folder with 1.6k images over NFS](https://github.com/ggand0/viewskater/issues/73)

## Problem

The application UI freezes when opening directories with many images over slow network filesystems (NFS). The root cause was synchronous `fs::read_dir()` calls blocking the main UI thread while enumerating directory contents.

**Affected code path**:
```
Message::FolderOpened → app.initialize_dir_path() → pane.initialize_dir_path()
  → file_io::get_image_paths() → fs::read_dir() ← BLOCKING
```

## Solution

Split directory initialization into two phases:
1. **Async Phase**: Directory enumeration runs in background via `Task::perform()` using `tokio::fs::read_dir()`
2. **Sync Phase**: When enumeration completes, `DirectoryEnumerated` message triggers sync initialization with pre-enumerated paths

## Implementation Details

### New Types (`src/app/message.rs`)

```rust
/// Result of async directory enumeration
pub struct DirectoryEnumResult {
    pub file_paths: Vec<PathBuf>,
    pub directory_path: String,
    pub initial_index: usize,
}

/// Error type for async directory enumeration
pub enum DirectoryEnumError {
    NoImagesFound,
    DirectoryError(String),
    NotFound,
}

// New message variant
DirectoryEnumerated(Result<DirectoryEnumResult, DirectoryEnumError>, usize),
```

### Async Enumeration Function (`src/file_io.rs`)

```rust
pub async fn enumerate_directory_async(path: PathBuf) -> Result<DirectoryEnumResult, DirectoryEnumError> {
    use tokio::fs as async_fs;

    // Determine if path is file or directory (sync metadata check is fast)
    let (dir_path, is_file_drop) = if is_file(&path) {
        let parent = path.parent().ok_or(DirectoryEnumError::NotFound)?.to_path_buf();
        (parent, true)
    } else if is_directory(&path) {
        (path.clone(), false)
    } else {
        return Err(DirectoryEnumError::NotFound);
    };

    // Async directory enumeration
    let mut entries = async_fs::read_dir(&dir_path).await
        .map_err(|e| DirectoryEnumError::DirectoryError(e.to_string()))?;

    let mut image_paths: Vec<PathBuf> = Vec::new();
    while let Some(entry) = entries.next_entry().await...? {
        // Filter by supported extensions
        if is_supported_extension(extension) {
            image_paths.push(entry_path);
        }
    }

    // Sort and return
    alphanumeric_sort::sort_path_slice(&mut image_paths);
    Ok(DirectoryEnumResult { file_paths: image_paths, ... })
}
```

### Modified Initialization Flow (`src/app.rs`)

**Before** (blocking):
```rust
pub fn initialize_dir_path(&mut self, path: &PathBuf, pane_index: usize) -> Task<Message> {
    pane.initialize_dir_path(...);  // BLOCKS on fs::read_dir
    load_initial_neighbors(...)
}
```

**After** (async):
```rust
pub fn initialize_dir_path(&mut self, path: &PathBuf, pane_index: usize) -> Task<Message> {
    // Compressed files use sync path (typically local)
    if is_compressed_file(path) {
        return self.initialize_dir_path_sync(path, pane_index);
    }

    // Reset state, clear slider images
    self.reset_state(pane_index as isize);

    // Dispatch async enumeration
    Task::perform(
        file_io::enumerate_directory_async(path.clone()),
        move |result| Message::DirectoryEnumerated(result, pane_index)
    )
}

pub fn complete_dir_initialization(&mut self, result: DirectoryEnumResult, pane_index: usize) -> Task<Message> {
    // Called when DirectoryEnumerated message arrives
    pane.initialize_with_paths(...);
    load_initial_neighbors(...)
}
```

### New Pane Method (`src/pane.rs`)

Added `initialize_with_paths()` that accepts pre-enumerated `Vec<PathBuf>` instead of calling `get_image_paths()`:
- Creates `ImageCache` with provided paths
- Loads initial image synchronously for immediate display
- Sets up Scene
- Returns control to caller for async neighbor loading

### Message Handler (`src/app/message_handlers.rs`)

```rust
Message::DirectoryEnumerated(result, pane_index) => {
    match result {
        Ok(enum_result) => app.complete_dir_initialization(enum_result, pane_index),
        Err(DirectoryEnumError::NoImagesFound) => { error!("..."); Task::none() }
        Err(DirectoryEnumError::DirectoryError(e)) => { error!("..."); Task::none() }
        Err(DirectoryEnumError::NotFound) => { error!("..."); Task::none() }
    }
}
```

### CLI Task Handling Fix

The `update()` function was discarding tasks from CLI path initialization. Fixed to properly batch and return async tasks:

```rust
fn update(&mut self, message: Message) -> Task<Message> {
    let mut cli_tasks: Vec<Task<Message>> = Vec::new();
    while let Ok(path) = self.file_receiver.try_recv() {
        let init_task = self.initialize_dir_path(&PathBuf::from(path), 0);
        cli_tasks.push(init_task);  // Collect instead of discard
    }

    // ... at end of function ...
    if !cli_tasks.is_empty() {
        cli_tasks.push(task);
        Task::batch(cli_tasks)
    } else {
        task
    }
}
```

## Files Modified

| File | Changes |
|------|---------|
| `src/app/message.rs` | Added `DirectoryEnumResult`, `DirectoryEnumError`, `DirectoryEnumerated` |
| `src/file_io.rs` | Added `enumerate_directory_async()` |
| `src/app.rs` | Split `initialize_dir_path()`, added `complete_dir_initialization()`, `initialize_dir_path_sync()`, extracted helpers |
| `src/pane.rs` | Added `initialize_with_paths()`, extracted `setup_scene_for_image()` helper |
| `src/app/message_handlers.rs` | Added handler for `DirectoryEnumerated` |

## Refactoring

After initial implementation, extracted helper methods to reduce code duplication:

### `src/app.rs` Helpers

```rust
/// Ensure panes vector has at least `pane_index + 1` panes
fn ensure_pane_exists(&mut self, pane_index: usize)

/// Clear cached slider images from all panes
fn clear_slider_images(&mut self)

/// Start loading neighbor images after directory initialization
/// Sets last_opened_pane, loads selection state, calls load_initial_neighbors()
fn start_neighbor_loading(&mut self, pane_index: usize) -> Task<Message>
```

These helpers are used by both `initialize_dir_path()` (async path) and `initialize_dir_path_sync()` (archive path).

### `src/pane.rs` Helper

```rust
/// Set up scene with cached image data (GPU texture, BC1 compressed, or CPU bytes)
fn setup_scene_for_image(&mut self, cached_data: &CachedData)
```

Used by `render_next_image()`, `render_prev_image()`, `initialize_dir_path()`, and `initialize_with_paths()`.

## Design Decisions

1. **Compressed files (zip/rar/7z) stay synchronous**: Archives are typically local, not over NFS
2. **macOS sandbox**: Async path works for standard access; sync fallback handles permission dialogs
3. **No loading indicator**: Deferred to separate branch
4. **Tokio**: Already available in Cargo.toml with `fs` feature

## Testing

### Local Test
Tested with 4318 images directory - async enumeration completes and images load correctly.

### NFS Latency Simulation (Docker + tc)

Used Docker container running NFS server with `tc` network delay to simulate slow NFS:

```bash
# 1. Start NFS server container with image directory mounted
docker run -d --name nfs-test --privileged \
  -v /home/gota/ggando/rust_gui/data/demo/small_images:/images \
  -e SHARED_DIRECTORY=/images \
  -p 2049:2049 \
  itsthenetwork/nfs-server-alpine

# 2. Add 50ms network delay to container's interface
docker exec nfs-test tc qdisc add dev eth0 root netem delay 50ms

# 3. Get container IP
docker inspect nfs-test --format '{{.NetworkSettings.IPAddress}}'
# Output: 172.17.0.2

# 4. Mount NFS share on host
sudo mkdir -p /mnt/nfs-test
sudo mount -t nfs -o vers=4 172.17.0.2:/ /mnt/nfs-test

# 5. Run ViewSkater with NFS path
RUST_LOG=viewskater=debug cargo run -- /mnt/nfs-test

# 6. Clean up
sudo umount /mnt/nfs-test
docker stop nfs-test && docker rm nfs-test
```

### Test Results

```
09:15:56.526 - Task queued (async enumeration started)
09:15:56.587 - Directory enumerated: 4318 images (61ms)
09:15:56.604 - Initial image displayed (~78ms total)
09:15:57.246 - Cache window loaded (11 images in 640ms, ~60ms/image with delay)
```

- **UI remained responsive** during directory enumeration
- Directory enumeration completed in 61ms for 4318 files
- First image displayed within 78ms of task dispatch
- Network delay visible in image loading times (~60ms/image vs ~20ms normally)
