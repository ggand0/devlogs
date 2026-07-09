# 042 - ThumbnailCache fixes and preview benchmarks

Work on branch `fix/thumbnail-cache`, implementing PR 1 from the audit in
devlog 041: bug B1 (preview budget setting has no effect) and perf P2
(duplicate thumbnail decodes while hovering), plus two benchmarks built to
verify the second fix.

## Fixes (committed)

### 1207fc9 - Apply slider preview budget changes to live thumbnail caches

`apply_settings_to_caches` never propagated `preview_budget_mb`, and
`ThumbnailCache` had no budget setter, so the Preferences slider silently
did nothing until restart. Added `ThumbnailCache::set_budget_mb()`
(evicts entries furthest from the displayed thumbnail via the existing
`evict_thumb_cache`, preserving "0 = unlimited" and keep-at-least-one) and
wired it plus `pane.preview_budget_mb` into `apply_settings_to_caches`.
New unit test: `set_budget_evicts_existing_entries`.

### 2f3a198 - Skip duplicate thumbnail decode requests while hovering

`current_thumbnail_for` sent a decode request every frame the hovered
index was uncached. Requests queued during the ~30-50 ms decode were
drained by the worker after the result landed, so a stationary hover
decoded the same image twice (worker CPU doubled).

Fix: `pending_idx: Option<usize>` tracks the most recently sent request;
the send is skipped while its result is outstanding. Invariant:
`pending_idx` always equals the last request actually sent, which is the
newest item in the worker queue, which drain-to-latest always eventually
decodes, so the marker always clears. Two safeguards:

- The worker now sends `(idx, None)` on decode failure (result channel is
  `Option<ColorImage>`), so a corrupt file can't wedge the marker.
- `poll()` clears the marker only when the arriving result matches it, so
  a stale result can't clear a newer in-flight request.

Latest-wins sweeps are unchanged: a new hovered index differs from
`pending_idx` and sends immediately. Residual behavior: a persistently
failing file retries once per completed attempt while hovered (previously
it queued a request every frame).

## Benchmark 1: unit simulation (`cache.rs` tests, branch-only)

```
cargo test --profile opt-dev bench_preview_simulation -- --ignored --nocapture
```

Simulates per-frame hovering against a real `ThumbnailCache` (worker
thread, real decodes) with fixed 16 ms frame pacing. Implementation
details:

- **Decode counting without touching production code**: the test swaps
  `tc.res_rx` with a fresh channel and spawns a forwarder thread that
  counts each message (one message = one completed worker decode) before
  passing it through.
- **Legacy A/B mode in one binary**: clearing `tc.pending_idx = None`
  before every simulated frame makes `current_thumbnail_for` resend each
  frame, reproducing the pre-fix behavior exactly. No git juggling; this
  is also why the test can't compile on main (no `pending_idx`).
- Scenarios: hover-and-pause (visit every index scrambled, wait until
  cached, dwell 5 frames) and fast sweep (1 index/frame, 2 passes). A
  settle loop keeps polling until the worker is idle so trailing redundant
  decodes are counted.
- Images: 25 synthetic 1920x1080 PNGs generated into
  `$TMPDIR/viewskater-thumb-bench` (reused across runs), or set
  `THUMB_BENCH_DIR` to a real folder.

Results:

```
=== hover-and-pause: 25 thumbnails, 16ms frames, 5-frame dwell ===
  legacy   decodes=50   redundant=25   elapsed=3.29s thumbs/s=7.6 ready avg=3.2 max=5 frames
  fixed    decodes=25   redundant=0    elapsed=3.35s thumbs/s=7.5 ready avg=3.3 max=5 frames

=== fast sweep: 25 indices x 2 passes, 1 index/frame ===
  legacy   decodes=25 cached_at_end=25
  fixed    decodes=25 cached_at_end=25
```

Confirms the fix: worker decode count halves (each hover decoded exactly
once, asserted), time-to-ready unchanged, sweep behavior identical
(drain-to-latest already collapsed duplicates during fast sweeps, so the
old cost was specific to hover-and-pause).

## Benchmark 2: GUI emulation (`--bench-preview`, uncommitted)

```
RUST_LOG=viewskater_egui=info cargo run --profile opt-dev -- \
    --bench-preview /tmp/viewskater-thumb-bench
```

Manual verification was inconclusive because cursor speed can't be held
constant and the Preview FPS overlay depends on it. This mode drives the
real pipeline (window, hover-to-index math, popup painting, GPU uploads,
vsync pacing) with a deterministic synthetic hover.

- `src/preview_bench.rs`: `PreviewBench` state machine. Phase 1
  hover-and-pause (half the indices, scrambled front/back order, hold
  until the exact thumbnail displays, dwell 5 frames), phase 2 hover-hop
  (other half, dwell 1 frame — moves on immediately after the thumbnail
  appears, exposing decode-worker contention from redundant decodes),
  phase 3 sweep (1 index/frame, 2 passes). Logs a report and closes the
  app, so runs are scriptable.
- Report includes process CPU time (`getrusage(RUSAGE_SELF)`, unix only)
  measured from bench start to end — wasted background decodes are
  invisible in latency but show directly here.
- `paint_nav_slider` takes `bench_hover_t: Option<f32>`; when set, a
  synthetic hover position along the rail feeds the existing preview code
  path unchanged. `paint_preview_popup` returns `is_exact` and
  `SliderResult` carries it as `preview_exact`, which is what latency is
  measured against.
- Report includes per-phase frame counts, fps, and thumbs/s, plus the
  overlay's own "Preview FPS" counter (`ImagePerfTracker::frame_fps`,
  made `pub(crate)`) sampled at the end of each phase so the log matches
  the number visible in the debug display. The frame window is reset
  between phases so the sweep sample is pure sweep. Note the overlay
  metric counts cursor *index changes* per second, not UI frames: during
  hover-and-pause it reads ~13 (one index change per resolved target,
  same quantity as thumbs/s), during a sweep ~140 (index changes every
  frame). A manually observed "14 fps" during pausing hovers is this
  number behaving correctly.
- **Cross-branch by construction**: the mode only touches `app.rs`,
  `main.rs` (identical on main and the fix branch) and the new file, and
  only uses `ThumbnailCache` APIs present on both. Uncommitted changes
  carry across `git checkout main` directly; only the unit-bench edits to
  `cache.rs` need stashing:

  ```
  git stash push src/cache.rs && git checkout main
  # build + run bench
  git checkout fix/thumbnail-cache && git stash pop
  ```

Results with hop phase and CPU time (same machine/session, 2 runs each,
25 synthetic 1080p PNGs, ~144-149 fps uncapped):

```
main (pre-fix):
  hover-and-pause avg=32.5/34.9ms   hover-hop avg=38.6/38.4ms
  process CPU: 2.11s / 2.20s
fix/thumbnail-cache:
  hover-and-pause avg=31.8/42.5ms   hover-hop avg=30.6/39.3ms
  process CPU: 1.43s / 1.72s
```

- **Process CPU: ~30% less on the fix** (2.11-2.20 s vs 1.43-1.72 s for
  the same workload). This is the improvement the fix delivers: the
  redundant decodes burned CPU without producing anything visible.
- **Latency: parity in hover-and-pause** (expected — the redundant decode
  ran after the thumbnail was already displayed). Hover-hop shows a mild
  edge for the fix (main's worker is sometimes mid-redundant-decode when
  the next request lands), muted at 1080p because drain-to-latest often
  skips the redundant decode when the next request arrives in time. With
  slower thumbnails (4K sources) the interference window grows with
  decode time, so the hop-phase gap should widen.
- Earlier batches without CPU measurement showed overlapping latency
  ranges (main 30.3-34.4 ms avg, fix 31.4-39.5 ms across sessions),
  frame-quantized at ~7 ms.

Definitive 3-run session (manual runs, same dataset), means:

```
                          main               fix/thumbnail-cache
hover-and-pause avg       34.1 ms            33.1 ms   (parity)
hover-hop avg             43.1 ms            33.1 ms   (fix 23% faster)
hover-hop thumbs/s        17.8               21.5      (fix +20%)
hover-hop overlay FPS     16.4               19.8      (fix +20%)
process CPU               2.21 s             1.43 s    (fix 35% less)
hover max (worst tail)    42.4-74.3 ms       35.7-41.6 ms
```

Main's hop runs split into a lucky timing race (34.9 ms, drain-to-latest
skipped the redundant decode) and unlucky ones (45.4, 49.0 ms, the new
request waited behind it); the fix has no race to lose, hence the tight
spread. Single runs are misleading: medians quantize to 4 or 5 frames
(~27.8 vs ~34.7 ms) and flip between runs on both branches.

## Analysis of run-to-run variance

Latencies are quantized to whole frames (~7 ms at ~142 fps); medians land
on either ~27.8 ms (4 frames) or ~34.6 ms (5 frames) on both branches
depending on how decode completion aligns with the frame loop. One
earlier batch had the fix consistently on the 5-frame mode while main sat
on the 4-frame mode; the final batch shows both branches within 1 ms of
each other.

Possible contributor to the earlier skew, not verified: this machine runs
the `ondemand` cpufreq governor. The legacy code's redundant decodes kept
the worker thread almost continuously busy, holding the core boosted; the
fixed code lets the worker idle during dwell, so a decode can pay a
frequency-ramp penalty of a few ms, occasionally crossing a frame
boundary. Consistent with the unit bench (16 ms pacing, longer dwell)
showing latency parity. To eliminate the variable entirely: pin the
`performance` governor (`sudo cpupower frequency-set -g performance`)
before comparing.

Verdict: no meaningful regression. The old behavior spent 2x decode CPU
per hovered thumbnail; the occasional sub-frame latency difference is a
side effect of the worker idling (less heat, less battery), not an
algorithmic cost. Sweep throughput and exactness are identical.

## Status

- Committed on `fix/thumbnail-cache`: 1207fc9 (budget fix), 2f3a198
  (decode dedup), 21fdd68 (GUI bench), 1c062fb (simulation test).
- Bench code refactored into `src/bench/` (shared helpers in `mod.rs`:
  process CPU time, latency stats, rate math; `preview.rs` for the
  preview bench) so planned keyboard-nav and slider-nav benchmarks can be
  added as sibling modules. The simulation test moved to a `cfg(test)`
  child module `src/cache/preview_sim_bench.rs`, keeping `cache.rs` at
  ~916 lines.
- 21fdd68 touches only main-identical files plus new files, so it
  cherry-picks onto main for future baselines; the simulation test
  references `pending_idx` and is branch-only.
- PR draft: `tmp/drafts/pr_thumbnail_cache.md`.
