# 017 Crash Logging and Log Buffer

Branch: `feat/logging`

## Summary

Added crash logging infrastructure: a composite logger that captures app logs in a circular buffer, a panic hook that dumps the buffer and backtrace to disk, and a "Show Logs" menu item to access the log directory.

## Implementation

### Composite logger (`file_io.rs`)

Two loggers combined into one via `CompositeLogger` implementing `log::Log`:

1. **`env_logger::Logger`** — console output to stderr. Default filter: `viewskater_egui=info`. Overridden by `RUST_LOG` env var.
2. **`BufferLogger`** — in-memory circular buffer. `VecDeque<String>` behind `Arc<Mutex<>>`, capped at 1000 entries. Only captures logs from targets starting with `viewskater_egui` at debug level or below.

```rust
pub fn setup_logger() -> Arc<Mutex<VecDeque<String>>>
```

Returns the shared buffer handle for the panic hook to use.

### Panic hook (`file_io.rs`)

Registers `std::panic::set_hook()` that writes to `panic.log`:
1. Panic message
2. `std::backtrace::Backtrace::force_capture()` (no external crate needed, stable since Rust 1.65)
3. Last 1000 log entries from the circular buffer

```rust
pub fn setup_panic_hook(log_buffer: Arc<Mutex<VecDeque<String>>>)
```

Uses `force_capture()` instead of `capture()` to ensure backtraces are always generated regardless of `RUST_BACKTRACE` env var.

### Log directory

```rust
pub fn get_log_directory() -> PathBuf {
    dirs::data_dir().unwrap_or_else(|| PathBuf::from(".")).join("viewskater-egui").join("logs")
}
```

- Linux: `~/.local/share/viewskater-egui/logs/`
- macOS: `~/Library/Application Support/viewskater-egui/logs/`
- Windows: `%APPDATA%/viewskater-egui/logs/`

### Show Logs menu item

Help > Show Logs creates the log directory (if missing) and opens it in the system file explorer via platform-specific commands (`xdg-open`, `open`, `explorer`).

### Initialization (`main.rs`)

```rust
let log_buffer = file_io::setup_logger();
file_io::setup_panic_hook(log_buffer);
```

Replaces the previous `env_logger::init()` call.

## Current behavior

- Logs are only written to disk on panic (backtrace + buffered entries → `panic.log`)
- Normal runtime logs go to stderr only
- The circular buffer is always active regardless of log level settings

## Files changed

- `src/file_io.rs` — `BufferLogger`, `CompositeLogger`, `setup_logger()`, `setup_panic_hook()`, `get_log_directory()`, `open_in_file_explorer()`
- `src/main.rs` — Replaced `env_logger::init()` with `setup_logger()` + `setup_panic_hook()`
- `src/menu.rs` — Added `ShowLogs` to `MenuAction`, "Show Logs" item under Help menu
- `src/app/handlers.rs` — Handle `MenuAction::ShowLogs`
