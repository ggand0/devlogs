# 018 Log Export

Branch: `feat/logging`

## Summary

Added "Export debug logs" menu item under Help that dumps the in-memory log buffer to `debug.log` on demand. Removed the continuous `FileLogger` that wrote to disk on every log call — unnecessary I/O for a GUI app. Now matches the iced version's architecture (PR #48).

## Architecture

Rust's `log` crate uses a global logger set once via `log::set_boxed_logger()`. Since only one global logger is allowed, `CompositeLogger` wraps two backends so that every `log::info!()`, `log::debug!()`, etc. call fans out to both:

```
log::info!("message")
        │
        ▼
  CompositeLogger
   ├── env_logger (console/stderr, for development)
   └── BufferLogger (in-memory VecDeque<String>, 1000 entry cap)
```

- **`env_logger`** — standard Rust console logger. Respects `RUST_LOG` env var. Default filter: `viewskater_egui=info`. Only useful when running from a terminal.
- **`BufferLogger`** — captures log output into a `VecDeque<String>` behind `Arc<Mutex<>>`. Circular buffer, oldest entries evicted when full. Filtered to `viewskater_egui` target at Debug level. This buffer is the single source for both the panic hook and on-demand export.

The `Arc<Mutex<VecDeque<String>>>` is shared three ways:
1. `CompositeLogger` (global logger) — writes to it on every log call
2. Panic hook — reads it to dump the last 1000 entries into `panic.log`
3. `App.log_buffer` — reads it when user clicks "Export debug logs"

### Log files

| File | When written |
|---|---|
| `debug.log` | On demand: Help > Export debug logs |
| `panic.log` | Automatically on panic (backtrace + last 1000 log entries) |

Log directory: `~/.local/share/viewskater-egui/logs/` (Linux), `~/Library/Application Support/viewskater-egui/logs/` (macOS), `%APPDATA%/viewskater-egui/logs/` (Windows).

## Iced version reference (PR #48)

The iced version has the same `BufferLogger` + `env_logger` + `CompositeLogger` pattern. It additionally implements stdout capture via `libc::pipe`/`libc::dup2` to intercept `println!` output, but this was evaluated and rejected for the egui version:

- `dup2` requires `unsafe` and violates Rust's I/O safety model
- Windows GUI subsystem apps have no stdout to capture
- macOS .app bundles bind stdout to `/dev/null`
- No established Rust GUI app uses `dup2` for stdout capture
- The correct fix is using `log::*` macros instead of `println!`

The iced version's "Export all logs" (debug + stdout) was dropped. Only "Export debug logs" is implemented.

## Decisions

- **No continuous file logging.** The iced version doesn't have it. Writing to disk on every log call is wasteful for an image viewer that runs for hours. The in-memory buffer + on-demand export covers all debugging scenarios.
- **No stdout capture.** `libc::dup2` is unsafe, cross-platform fragile, and solves a problem that shouldn't exist (using `println!` instead of `log::*`).
- **Buffer on `App` struct, not global statics.** The iced version uses `once_cell::Lazy` statics with `Arc<Mutex<Option<...>>>` wrappers. The egui version passes the `Arc` through `App::new()` directly — simpler, no global state.

## Commits

1. `92e8c91` Add crash logging and Show Logs menu item
2. `e1cd4fc` Add persistent debug.log file logging
3. `e4091e3` Add Export debug logs menu item and remove FileLogger
