# Slider navigation benchmark: --bench-slider

Date: 2026-09-14
Context: second half of plan 010, on branch feat/nav-bench after the
keyboard bench (devlog 052). Commit 991996c on top of the five
keyboard-bench commits.
Numbers from gota-home, warm page cache, default settings, 4k_PNG_10MB
(140 images). Not verified claims are marked as such.

## What it does

`viewskater-egui <folder> --bench-slider` injects a synthetic drag into
`paint_nav_slider` (`BenchDrag { t, released }`, next to the preview
bench's `bench_hover_t`). Everything downstream runs as for a real
drag: target index from the rail position, `apply_slider_target` with
its 10 ms `SliderLoader` throttle and main-thread `load_sync`,
`apply_slider_release` recentering the sliding window. Thumbnails are
forced off while the bench runs, as they are during a real drag.

Phases, after the same settle wait as the keyboard bench:

1. Sweep. t from 0 to 1 over `--bench-sweep-secs` (default 4), one
   position per frame, release, wait for the window to refill, then
   back from 1 to 0 and release again. Both directions, both refills.
2. Scrub. `--bench-scrub-anchors` (default 3) positions spaced evenly
   along the rail, so three anchors sit at 25, 50 and 75 percent of the
   folder. At each: press, drag back and forth across
   `--bench-scrub-span` of the rail (default 0.2, a tenth each side),
   `--bench-scrub-passes` times (default 3) over `--bench-scrub-secs`
   (default 2), release, wait for refill. A person hunting for a frame;
   the return passes revisit what was just loaded, so the decode LRU
   shows here. The first version moved 5 percent once at the two ends of
   the folder, because the anchors came from the preview bench's
   scrambled order; the owner rejected both.
3. Jump. `--bench-jumps` (default 20) targets. Candidates are twice as
   many evenly spaced positions over the whole rail, minus any inside a
   scrubbed region, visited first, last, second, second to last so
   consecutive clicks are far apart. With three anchors and a 0.2 span,
   60 percent of the rail is scrubbed and only 16 candidates remain, so
   the default run does 16 jumps. Lower the span or the anchor count to
   get 20. Targets stay out of scrubbed regions because otherwise they
   hit images the scrub just put in the LRU (16 of 20 in the first run)
   and the phase measures cache hits instead of clicks.

Per phase: how many images the handle pointed at and how many of those
were displayed, sync loads with the time each blocked the frame (decode
and convert separately; the "upload" is egui queueing the copy and reads
0.0, so it is in the JSON but not printed), LRU hits, release-to-refill
time, click-to-image time for jumps, frame time p50/p99/max, background
decode times, CPU seconds, peak RSS and GPU memory.

## Reading the numbers

- "pointed at a new image on N frames (folder: M images)": every frame,
  the app converts the handle's position on the rail into an image
  number, the same way it does for the mouse. N counts the frames where
  that number was different from the image on screen; each of those
  frames asks for its image. In a scrub the same image is pointed at
  again on every pass, so N can exceed M. In a sweep the handle moves on
  the clock, so when a decode blocks a frame for 60 ms the handle has
  moved two or three images by the next frame, and the images in between
  are never asked for. That is why a sweep over 140 images asks for
  about 60.
- "displayed (x%)": of those asks, how many put the image on screen.
  The only way one does not: `SliderLoader::should_load` refuses a
  decode less than 10 ms after the previous one. 100% means it never
  refused. It says nothing about the images the handle moved past.
- "sync block": how long `load_sync` held the main thread per load,
  decode plus convert. This is the frame hitch a person feels while
  dragging.
- "refill": from release to a full sliding window and no decode thread
  alive. What the next key press would wait on.
- "click-to-image": jump phase only, from the click frame to the first
  frame whose current index is the target and has a texture. Includes
  the blocking decode inside the click frame.
- LRU hits are counted inside `load_sync`; a sliding-window hit in
  `apply_slider_target` never reaches `load_sync` and is not an LRU hit.

## Implementation details

Injection. `paint_nav_slider` takes `bench_drag: Option<BenchDrag>`.
When Some, the target index is computed from `drag.t` with the same
formula the pointer path uses (`round(t * (n - 1))`), the pointer is
ignored, and `released` is `drag.released` instead of
`response.drag_stopped()`. Nothing after that line knows the difference.
The two independent-mode sliders pass None. Thumbnails: the `preview`
flag in `show_slider_panel` is false while `slider_bench` is Some, the
same as `!response.dragged()` during a real drag; a real mouse resting
over the window cannot start thumbnail decodes into the measurement.

Frame order in `show_slider_panel`: compute `now` and `bench_drag`
from `slider_bench.drag(now)`, paint the slider (which applies the
drag), `apply_slider_result_all` (now returns whether any pane showed
an image), then `tick_slider_bench(now, bench_drag, target_changed,
shown)`. The bench sees the frame's own drag together with what it
caused, so positions and shown counts line up per frame.

State machine (`SliderBench`). Phases Settle, Sweep, Scrub, Jump, Done.
Within a phase a `Gesture`: `Dragging { since }` while the finger is
down; `Refilling { released_at }` after a release, until the frame
reports settled; `Landing { clicked_at, target }` for jumps, until the
frame reports the target on screen, then Refilling from the click time.
`drag(now)` is pure: sweep is `t = elapsed / sweep_secs` on the first
pass and `1 - t` on the return, released at the end of each; scrub is
`passes` repetitions of three linear legs (anchor, +span/2, -span/2,
anchor) over `secs`, clamped to the rail; jump is `{ t, released: true }`
every frame it is asked. A release returns early from `tick` so the frame
that shows the image is observed on the next tick, where the sync
decode has already run inside the release frame. Landing and Refilling
are checked in sequence in one tick because a click that hits the LRU
can land and be settled on the same frame. Anchors are `i / (k + 1)` for `k` anchors; jump targets are described
above.

Timeouts: refill or landing waits longer than 10 s mark the phase timed
out and move on; settle 60 s. Same constants as the keyboard bench.

Samples. `Pane::sync_samples: Option<(Vec<SyncSample>, usize)>` is Some
only between `set_sync_sampling(true)` and `(false)`; `load_sync`
pushes decode, convert and upload times on a miss and bumps the hit
count on an LRU hit. The driver drains it and the background
`decode_samples` sink at each `PhaseEnd` so samples land in the phase
that produced them. `start_slider_bench` clears the background sink
first, because the keyboard bench's last phase may have left decodes in
it. `upload_ms` is the `ctx.load_texture` call, which egui queues; it
reads 0.0 and is in the JSON only.

Run sequencing (`src/app/bench.rs`). `start_run(run_start)` turns on
decode sampling and starts nav if asked, else slider. `finish_nav_bench`
stashes `run_nav_report` and, if slider is asked, starts the slider
bench with `Instant::now()` as its settle origin (no reopen happened).
`tick_slider_bench` on Done calls `finish_run(Some(slider_report))`,
which assembles `BenchReport { header, nav, slider }`, prints to stderr,
writes JSON and markdown, pushes to `bench_reports`, and then repeats
the folder, moves to the next `--bench-dir`, or calls
`finish_all_benchmarks` for the summary. Settled means
`pane.is_settled() && active_decode_threads() == 0`, shared with the
keyboard bench through `pane_state()`.

Report. `SliderPhaseReport` per phase, `SliderReport` with the settle
times and phase parameters, `SyncStats` for the sync loads. Summary:
`SliderPhaseSummary` per phase per folder with `Spread` (mean, min, max)
of display ratio, sync block p50 and p95, LRU hits, refill p50, jump p50
and p95, frame p99, CPU. `Spread::cell` prints the range only when the
runs differ.

Tests (src/bench/slider.rs): scrambled anchors and jumps avoiding the
scrubbed ranges; sweep motion, release and refill accounting; scrub
legs and clamping at the rail end, then a fresh press at the next
anchor; jump landing then refill, two jumps; a wait that never ends
times out. Report tests cover the slider summary aggregation and the
markdown row.

## Results, 4k_PNG_10MB, one run, final defaults

| Phase | Frames pointing at a new image | Displayed | Sync block p50 / p95 ms | LRU hits | Refill p50 ms | Click-to-image p50 ms | Frame p50 / p99 ms |
|---|---|---|---|---|---|---|---|
| sweep, both ways | 129 | 127 (98%) | 63.0 / 70.4 (n=100) | 7 | 292 (2 releases) | | 7.2 / 117.6 |
| scrub x3 | 166 | 136 (82%) | 63.3 / 73.9 (n=69) | 67 | 262 (3 releases) | | 6.9 / 91.3 |
| jump x16 | 16 | 16 (100%) | 65.1 / 75.7 (n=16) | 0 | 308 (16 releases) | 178.1 | 7.0 / 190 |

Reading the sweep: two passes over 140 images pointed at 129 new
images in total, about 64 per pass. Each sync decode blocks the frame
for about 63 ms, the next frame reads the rail position 63 ms further
along, and the handle has moved two or three images; the ones in
between are never asked for. The return pass gets 7 LRU hits, not 64:
the 1 GB LRU budget holds about 31 4K textures (3840 x 2160 x 4 bytes
is 31.6 MB), so roughly 70 of the first pass's 100 sync-loaded images
were already evicted, and of the 31 still cached the return pass points
at every second one and gets the nearest from the sliding window.

Reading the scrub: 82 percent displayed is the 10 ms throttle refusing
loads. On LRU hits frames run at 7 ms, so every second request inside a
cached stretch is refused. Correct behaviour, invisible with the old
tiny scrub.

## Finding: every slider release decodes the current image again

Minor, but worth having on record. `SlidingWindowCache::jump_to`
(cache.rs line 466) is `initialize`, which does two separate things on
every `apply_slider_release`:

1. Decodes the center image synchronously with `decode_sync` on the
   main thread. That image is already on screen: `load_sync` just
   decoded it or fetched it from the LRU. This is the blocking,
   redundant part.
2. Drops every loaded slot and requests all neighbours again on the
   background threads. Not blocking, but wasteful when the window
   barely moved, and it is what the refill time measures.

The benchmark shows it directly: a click-to-image of 170 ms on 4K is
two 60 to 90 ms decodes of the same file back to back, the click's own
`load_sync` and the release's `initialize`. Even a click that hits the
LRU paid 93 ms in the first run, which is the redundant decode alone.
The 225 to 294 ms refill after every release is the whole window being
rebuilt from nothing.

Not changed in this branch; the benchmark PR should not alter app
behaviour. It is the obvious first before-and-after for the bench: make
`jump_to` keep the current texture (from the LRU or the pane) and keep
whichever loaded slots still fall inside the new window. Expected effect
on 4K: click-to-image halves, refill shrinks to the slots that actually
moved.

## Not done

- Runner scripts and the cold-cache option (plan 010 section 9).
- Baseline rows for FPS_BASELINES.md; macOS and Windows runs.
- Preview bench report is still its own text, not in the JSON.
- The `jump_to` fix above, as its own branch measured with this bench.
