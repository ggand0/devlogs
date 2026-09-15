# Slider navigation benchmark: --bench-slider

Date: 2026-09-14, updated 2026-09-15
Context: second half of plan 010, on branch feat/nav-bench after the
keyboard bench (devlog 052). Nine commits, 991996c to 1b3a02a, on top
of the five keyboard-bench commits; the branch has fourteen in all.
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
2. Scrub. `--bench-scrub-anchors` (default 5) positions spaced evenly
   from the first image to the last, so five anchors sit at 0, 25, 50,
   75 and 100 percent of the folder. At each: press, drag right by half
   of `--bench-scrub-span` (default 0.1, so 5 percent of the rail),
   left to the same distance before the anchor, back to the anchor,
   `--bench-scrub-passes` times (default 2) over `--bench-scrub-secs`
   (default 2), release, wait for refill. A person hunting for a frame;
   the return passes revisit what was just loaded, so the decode LRU
   shows here. History: the first version moved 5 percent once at ten
   anchors taken from the preview bench's scrambled order, which put
   them all at the two ends. A second version widened it to 10 percent
   each side, three passes, still at the ends. A third moved three
   anchors to 25, 50 and 75. The owner settled on the 5 percent motion
   with two passes at 0, 25, 50, 75 and 100.
3. Jump. `--bench-jumps` (default 20) positions spaced evenly over the
   rail, visited first, last, second, second to last so consecutive
   clicks are far apart. Each is a press and release in one frame.

Any of skate left, sweep, scrub and jump can be left out with
`--bench-skip skate-left,sweep,scrub,jump` (comma separated, any
subset). Skate right cannot be skipped, since skate left needs it to
reach the far end, and the tap phase is opt-in through
`--bench-tap-steps`. With the sweep skipped, settle goes straight to
the scrub; with everything skipped the slider bench reports settle only.

The phases are independent. The driver empties the decode LRU
(`Pane::clear_decode_lru`) when the slider bench starts and whenever a
phase ends, so nothing one phase loaded can turn the next phase's loads
into cache hits. History: the first version instead kept jump targets
out of the regions the scrubs had covered, which made the jump count
depend on the scrub flags (16 instead of 20 with wide scrubs) and was
the wrong fix; the owner pointed out the phases have nothing to do with
each other. LRU hits inside a phase are still counted and reported, and
now come only from that phase's own revisits. Emptying a 1 GB LRU frees
about 31 4K textures on the GPU at the next frame; that frame belongs
to the wait after the last release, not to a gesture.

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
  load less than 10 ms after the previous one. 100% means it never
  refused. It says nothing about the images the handle moved past. In
  practice a refusal follows an LRU hit: the hit costs nothing, the
  frame finishes in 7 ms, the next frame already points at another
  image, and it is inside the 10 ms. A sweep with 6 hits had 1 refusal,
  a scrub with 88 hits had 1; jumps never, the clicks are seconds apart.
  Small app detail: `should_load` is consulted before `load_sync` looks
  in the LRU, so a hit arms the throttle although it decoded nothing.
  Fix would be to check the LRU first. Not changed in this branch.
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
can land and be settled on the same frame. Anchors are `i / (k - 1)` for `k` anchors, so the first and last image
are always included; jump targets are `(i + 0.5) / jumps` in scrambled
order.

Timeouts: refill or landing waits longer than 10 s mark the phase timed
out and move on; settle 60 s. Same constants as the keyboard bench.

Recorded times. `Pane::sync_load_times: Option<(Vec<SyncLoadTiming>,
usize)>` is Some only between `record_sync_load_times(true)` and
`(false)`; `load_sync` pushes decode, convert and upload times on a
miss and bumps the hit count on an LRU hit. The driver drains it and the
cache's `decode_times_ms` list at each `PhaseEnd` so the times land in
the phase that produced them. `start_slider_bench` drops what the
keyboard bench's last phase left in the cache's list. `upload_ms` is the
`ctx.load_texture` call, which egui queues; it reads 0.0 and is in the
JSON only. Pane owns the cache, so the pane methods just forward to it.

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
| jump x16 (before the count fix) | 16 | 16 (100%) | 65.1 / 75.7 (n=16) | 0 | 308 (16 releases) | 178.1 | 7.0 / 190 |

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

## Finding: a throttled request leaves the index ahead of the picture

`apply_slider_target` sets `current_index` to the new image before it
asks `should_load`. When the throttle refuses, the index has moved and
the texture has not. On the next frame the slider's index equals the
pane's, so nothing is requested, and the refused image is never loaded
during the drag: the screen shows the previous picture under the new
position until the handle reaches the image after it. The release fixes
it, since `jump_to` decodes whatever `current_index` says. One frame or
so per refusal, and refusals are rare (one per sweep, one per scrub in
the runs above), so nobody has noticed. Fix: move the index only when a
texture was found or a load was made. Not changed in this branch.

## What the slider numbers say about the app

- Dragging through large images blocks the main thread on every
  uncached position. The throttle only spaces loads 10 ms apart, and a
  4K decode is 65 ms, so during a first pass the drag is a chain of
  blocking decodes: sync block p50 63 ms, frame p99 90 to 117 ms, frame
  max up to 190 ms. The throttle does nothing for heavy images; it only
  ever fires after a cache hit. The structural fix is to stop decoding
  on the slider path and request the image from a decode thread,
  showing the nearest available image until it lands. Its own branch,
  measured with this bench: sync block count should go to zero and
  frame p99 to the vsync interval.
- Every release decodes the current image again and rebuilds the whole
  window (finding above). Click-to-image 175 ms on 4K is two decodes.
- Displayed under 100 percent is the throttle rejecting a request that
  followed a cache hit within 10 ms. Rare, and it also causes the
  index-ahead-of-picture glitch above.

## Changes after the first version, 2026-09-14 to 2026-09-15

In order, each its own commit:

- Report counts renamed from an invented word to what is counted:
  "pointed at a new image on N frames (folder: M images), K displayed".
- Scrub widened to 10 percent each side, three passes, then set back to
  the original 5 percent motion with two passes, at 0, 25, 50, 75 and
  100 percent of the folder. Anchors had been at the two ends because
  they came from the preview bench's scrambled order.
- Sweep goes both ways, with a release and a refill wait at each end.
- Jump targets first avoided the scrubbed regions (which made the jump
  count depend on the scrub flags and gave 16 instead of 20), then the
  coupling was removed: the LRU is emptied when the slider bench starts
  and at every phase end, and jump targets are 20 evenly spaced
  positions in the far-apart order.
- `--bench-skip skate-left,sweep,scrub,jump`, any subset.
- Every scrub parameter is a flag: anchors, span, passes, seconds.
- Refactor before the PR (c7972ef): shared `PhaseStats` and `FrameTimer` in
  bench/phase.rs, the flags as a `BenchArgs` struct in the bench module
  flattened into the app's `Args`, one `BenchState` on `App` instead of
  eight fields, and the recorded-time lists named for what they hold
  (`decode_times_ms`, `sync_load_times`) instead of "samples". Net minus
  sixty lines, numbers unchanged.

## Not done

- Runner scripts from plan 010 section 9. `--bench-dir`, `--bench-runs`
  and the in-process summary do what `bench_nav.sh` was for; a compare
  script against main is worth writing when the first before-and-after
  branch needs it. The cold-cache option stays manual: drop caches by
  hand and tag the run with `--bench-label cold`.
- Baseline rows for FPS_BASELINES.md; macOS and Windows runs.
- Preview bench report is still its own text, not in the JSON.
- The three app findings above, each its own branch measured with this
  bench: `jump_to` rebuilding the window, the index moving on a refused
  load, and the synchronous decode on the slider path.
