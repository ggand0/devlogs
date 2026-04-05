# 018 Logging with tracing

Branch: `feat/logging`

## Summary

Replaced `env_logger` with `tracing` + `tracing-subscriber` for logging. Added crash logging (panic hook), "Show Logs" and "Export debug logs" menu items. Added compact debug-level logging for navigation and cache state.

## Architecture

```
log::info!("message")
        │
        │  (tracing-log bridge, automatic)
        ▼
tracing_subscriber::registry()
 ├── fmt::layer() + EnvFilter    → console/stderr
 └── BufferLayer                 → in-memory VecDeque<String>
```

### Two layers

1. **`fmt::layer()`** — tracing-subscriber's built-in formatted console output. Replaces `env_logger`. Respects `RUST_LOG` env var, default filter `viewskater_egui=info`. Colored output with timestamps, level, and target.

2. **`BufferLayer`** — custom `tracing_subscriber::Layer` implementation. Captures events from `viewskater_egui` target at debug level into a `VecDeque<String>` behind `Arc<Mutex<>>`. Circular buffer, 1000 entry cap, oldest evicted when full. No `EnvFilter` — filtering is done manually in `on_event` via the `log.target` field (see tracing-log caveat below).

### Why tracing instead of log + env_logger

The `log` crate only allows one global logger via `log::set_boxed_logger()`. To fan out to multiple destinations (console + buffer), the previous approach required a hand-rolled `CompositeLogger` that manually dispatched to each backend.

`tracing-subscriber` has native layer composition via `registry()`. Multiple layers are a first-class feature — you just `.with()` each one. No manual composite needed.

The existing `log::info!()`, `log::debug!()`, etc. calls throughout the codebase still work because `tracing-subscriber` includes a `tracing-log` bridge that automatically forwards `log` crate events into the tracing system.

### tracing-log caveat

When `log::info!()` events are forwarded through the tracing-log bridge, `event.metadata().target()` is `"log"` — NOT the original log target like `"viewskater_egui::file_io"`. The real target is stored in the `"log.target"` event field. `MessageVisitor` extracts this and `BufferLayer::on_event` uses it for filtering.

Also: calling `log::*` macros while holding the buffer lock causes a deadlock (the log call re-enters `BufferLayer::on_event` which tries to lock again). The export function drops the lock before calling `log::info!()`.

### Buffer sharing

The `Arc<Mutex<VecDeque<String>>>` is shared three ways:
1. **`BufferLayer`** (inside tracing registry) — writes on every log event
2. **Panic hook** — reads to dump last 1000 entries into `panic.log`
3. **`App.log_buffer`** — reads when user clicks "Export debug logs"

Passed from `main()` → `setup_panic_hook()` (cloned) and `App::new()`.

### MessageVisitor

tracing events carry structured fields, not plain strings. `BufferLayer` uses a `MessageVisitor` implementing `tracing::field::Visit` to extract the `message` and `log.target` fields from each event. It handles both `record_str` (string literals) and `record_debug` (formatted values via `Display`/`Debug`).

## Menu items

- **Help > Show Logs** — opens log directory in file explorer (`xdg-open`/`open`/`explorer`)
- **Help > Export debug logs** — dumps in-memory buffer to `debug.log`, then opens log directory

## Log files

| File | When written |
|---|---|
| `debug.log` | On demand via Help > Export debug logs |
| `panic.log` | Automatically on panic (backtrace + last 1000 buffer entries) |

Log directory: `~/.local/share/viewskater-egui/logs/` (Linux), `~/Library/Application Support/viewskater-egui/logs/` (macOS), `%APPDATA%/viewskater-egui/logs/` (Windows).

## Debug logging for navigation and cache

All navigation logging is **debug level only** — appears with `RUST_LOG=viewskater_egui=debug`, silent otherwise. No UI toggle needed.

### Arrow key / skating navigation

```
nav → 42/226 cache=[40..50] 10/11 hit
nav ← 41/226 cache=[39..49] 10/11 hit
```

- `→` / `←` — navigation direction
- `42/226` — current file index / total images
- `cache=[40..50]` — file index range covered by the sliding window cache
- `10/11` — loaded slots / total slots in the window
- `inflight=N` — (shown if > 0) number of background decode threads in progress
- `hit` / `miss` — whether the target image was already in the cache

### Home/End jump

```
jump 0/226 cache=[0..10] 1/11 miss
```

Same fields as nav. `miss` means the image wasn't in the cache window (fell back to `load_sync`).

### Slider drag (synchronous decode)

```
load_sync [608] (1024x791): decode=3.8ms convert=0.6ms cache=0.3ms upload=0.0ms total=4.7ms [LRU: 50]
LRU hit [428]: clone=0.9ms upload=0.0ms
```

**`load_sync`** — full decode path (cache miss during slider drag):
- `[608]` — file index
- `(1024x791)` — decoded image pixel dimensions
- `decode` — `image::open()` disk read + decompress time
- `convert` — `DynamicImage` → `egui::ColorImage` pixel format conversion
- `cache` — inserting decoded image into LRU decode cache
- `upload` — `TextureHandle::set()` GPU texture update (0ms because egui defers actual GPU upload)
- `total` — wall time for the full operation
- `[LRU: 50]` — current entry count in the LRU decode cache

**`LRU hit`** — revisiting a previously decoded image during slider drag:
- `clone` — cloning the `ColorImage` from the LRU cache
- `upload` — texture update

### Slider release

```
slider release 707/1000 cache=[702..712] 1/11 inflight=10
```

Logged when the slider drag ends and the cache re-centers around the new position:
- `1/11` — only 1 slot loaded (the current image via `load_sync`)
- `inflight=10` — 10 background threads spawned to decode neighbors

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
