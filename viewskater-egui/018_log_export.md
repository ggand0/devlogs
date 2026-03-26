# 018 Log Export

Branch: `feat/logging`

## Summary

Added "Export debug logs" and "Export all logs" menu items under Help, matching the iced version's PR #48 (`feat/log-export`). Currently only the debug buffer export is implemented; stdout capture via `libc::dup2` pipe redirection is the remaining piece for "Export all logs".

## Iced version reference (PR #48)

The iced version implements two distinct log sources:

1. **Debug buffer** — in-memory `VecDeque<String>` behind `Arc<Mutex<>>`, captures output from `log` macros (`debug!`, `info!`, `warn!`, `error!`), 1000 entry cap. Exported to `debug.log`.
2. **Stdout buffer** — captures `println!` output by redirecting stdout through a Unix pipe (`libc::pipe` + `libc::dup2`). A background thread reads from the pipe read-end, writes to the original stdout (so console still works), and appends timestamped entries to a separate `VecDeque`. Exported to `stdout.log`. Has a `#[cfg(not(unix))]` fallback that provides a manual-only capture buffer on Windows.

Global access pattern: `once_cell::sync::Lazy` statics (`SHARED_LOG_BUFFER`, `SHARED_STDOUT_BUFFER`) wrapped in `Arc<Mutex<Option<Arc<Mutex<VecDeque<String>>>>>>`, set during `main()` init, accessed from message handlers via `get_shared_log_buffer()` / `get_shared_stdout_buffer()`.

Menu items:
- "Export debug logs" → `Message::ExportDebugLogs` → `file_io::export_and_open_debug_logs()`
- "Export all logs" → `Message::ExportAllLogs` → `file_io::export_and_open_all_logs()` (exports both debug + stdout buffers)

Export file format: header block with `================` separators, export timestamp, entry count, then all buffer entries. Uses `chrono::Utc` for timestamps.

### Stdout capture mechanism (`setup_stdout_capture`)

```
pipe() → [read_fd, write_fd]
dup(STDOUT_FILENO) → original_stdout_fd
dup2(write_fd, STDOUT_FILENO)  // redirect stdout to pipe
close(write_fd)

thread::spawn → {
    loop {
        BufReader::read_line(read_fd)
        write to original_stdout (console passthrough)
        append timestamped entry to STDOUT_BUFFER
    }
}
```

Key detail: the original stdout fd is dup'd before redirection so console output still works. The capture thread reads from the pipe and both passes through to console and captures to buffer.

## Egui version implementation

### What's done

1. **`export_debug_logs()`** — dumps in-memory buffer to `debug_export.log` with formatted header/footer/timestamp using `chrono::Local`. Separate from the continuous `debug.log` written by `FileLogger`.
2. **`export_and_open_debug_logs()`** — calls `export_debug_logs()` then opens log directory.
3. **`export_and_open_all_logs()`** — currently calls `export_debug_logs()` only. Needs stdout capture to be complete.
4. **`MenuAction::ExportDebugLogs`** and **`MenuAction::ExportAllLogs`** — added to enum, menu items under Help, handlers in `app/handlers.rs`.
5. **`App.log_buffer`** — `Arc<Mutex<VecDeque<String>>>` stored on `App` struct, passed from `main()` through `App::new()`. The same `Arc` is cloned for the panic hook. This avoids the global static pattern the iced version uses.

### What's remaining

- **Stdout capture**: Implement `setup_stdout_capture()` with `libc::pipe` + `libc::dup2` pipe redirection (Unix) and a no-op fallback (Windows). Store the stdout buffer on `App` alongside the debug buffer.
- **`export_stdout_logs()`**: Write stdout buffer to `stdout.log` with the same header format.
- **Wire `export_and_open_all_logs()`** to export both buffers and open the directory.

### Log files in the directory

| File | Source | When written |
|---|---|---|
| `debug.log` | `FileLogger` (continuous) | Every log macro call |
| `debug_export.log` | `export_debug_logs()` (on-demand) | "Export debug logs" / "Export all logs" menu |
| `stdout.log` | `export_stdout_logs()` (on-demand) | "Export all logs" menu (once stdout capture is implemented) |
| `panic.log` | Panic hook | On panic only |

### Differences from iced version

- Egui version has continuous `debug.log` via `FileLogger` in addition to the on-demand export. Iced version only writes debug logs on demand.
- Egui version stores the buffer on `App` struct directly instead of global `Lazy` statics.
- Export uses `chrono::Local` (local time) instead of `chrono::Utc`.
- On-demand export file is `debug_export.log` (not `debug.log`) to avoid overwriting the continuous log.

## Files changed

- `Cargo.toml` — Added `chrono` and `libc` to `[dependencies]`
- `src/file_io.rs` — `export_debug_logs()`, `export_and_open_debug_logs()`, `export_and_open_all_logs()`
- `src/app.rs` — Added `log_buffer` field to `App`, updated constructor
- `src/main.rs` — Clone `log_buffer` for both panic hook and `App::new()`
- `src/menu.rs` — `ExportDebugLogs`, `ExportAllLogs` in `MenuAction`, menu items under Help
- `src/app/handlers.rs` — Handlers for the two new actions
