# 018 Logging with tracing

Branch: `feat/logging`

## Summary

Replaced `env_logger` with `tracing` + `tracing-subscriber` for logging. Added crash logging (panic hook), "Show Logs" and "Export debug logs" menu items.

## Architecture

```
log::info!("message")
        │
        │  (tracing-log bridge, automatic)
        ▼
tracing_subscriber::registry()
 ├── fmt::layer() + EnvFilter    → console/stderr
 └── BufferLayer + EnvFilter     → in-memory VecDeque<String>
```

### Two layers

1. **`fmt::layer()`** — tracing-subscriber's built-in formatted console output. Replaces `env_logger`. Respects `RUST_LOG` env var, default filter `viewskater_egui=info`. Colored output with timestamps, level, and target.

2. **`BufferLayer`** — custom `tracing_subscriber::Layer` implementation. Captures events from `viewskater_egui` target at debug level into a `VecDeque<String>` behind `Arc<Mutex<>>`. Circular buffer, 1000 entry cap, oldest evicted when full.

### Why tracing instead of log + env_logger

The `log` crate only allows one global logger via `log::set_boxed_logger()`. To fan out to multiple destinations (console + buffer), the previous approach required a hand-rolled `CompositeLogger` that manually dispatched to each backend.

`tracing-subscriber` has native layer composition via `registry()`. Multiple layers are a first-class feature — you just `.with()` each one. No manual composite needed.

The existing `log::info!()`, `log::debug!()`, etc. calls throughout the codebase still work because `tracing-subscriber` includes a `tracing-log` bridge that automatically forwards `log` crate events into the tracing system.

### Buffer sharing

The `Arc<Mutex<VecDeque<String>>>` is shared three ways:
1. **`BufferLayer`** (inside tracing registry) — writes on every log event
2. **Panic hook** — reads to dump last 1000 entries into `panic.log`
3. **`App.log_buffer`** — reads when user clicks "Export debug logs"

Passed from `main()` → `setup_panic_hook()` (cloned) and `App::new()`.

### MessageVisitor

tracing events carry structured fields, not plain strings. `BufferLayer` uses a `MessageVisitor` implementing `tracing::field::Visit` to extract the `message` field from each event. It handles both `record_str` (string literals) and `record_debug` (formatted values via `Display`/`Debug`).

## Menu items

- **Help > Show Logs** — opens log directory in file explorer (`xdg-open`/`open`/`explorer`)
- **Help > Export debug logs** — dumps in-memory buffer to `debug.log`, then opens log directory

## Log files

| File | When written |
|---|---|
| `debug.log` | On demand via Help > Export debug logs |
| `panic.log` | Automatically on panic (backtrace + last 1000 buffer entries) |

Log directory: `~/.local/share/viewskater-egui/logs/` (Linux), `~/Library/Application Support/viewskater-egui/logs/` (macOS), `%APPDATA%/viewskater-egui/logs/` (Windows).

## Dependencies changed

- Added: `tracing`, `tracing-subscriber` (with `env-filter` feature)
- Removed: `env_logger`
- Kept: `log` (still used by existing code and third-party crates)

## Previous approach (feat/logging-log-crate branch)

The first implementation used `log` + `env_logger` with a hand-rolled `CompositeLogger`:
- `CompositeLogger` implemented `log::Log`, manually dispatched to `env_logger` + `BufferLogger` + `FileLogger`
- `FileLogger` wrote to `debug.log` continuously on every log call (unnecessary disk I/O)
- Required `chrono` for timestamps in export headers, `libc` for planned stdout capture

All of that was replaced by the tracing approach which is simpler and idiomatic.
